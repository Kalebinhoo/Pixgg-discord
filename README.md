# 🎮 Integração PixGG em Bots Discord

Guia completo para integrar pagamentos PIX via **PixGG** em bots Discord usando Node.js e Axios.

---

## 📋 Índice

1. [O que é PixGG?](#o-que-é-pixgg)
2. [Configuração Inicial](#configuração-inicial)
3. [Endpoints Disponíveis](#endpoints-disponíveis)
4. [Implementação Básica](#implementação-básica)
5. [Fluxo de Pagamento Completo](#fluxo-de-pagamento-completo)
6. [Verificação de Pagamento](#verificação-de-pagamento)
7. [Consulta de Saldo](#consulta-de-saldo)
9. [Troubleshooting](#troubleshooting)

---

## O que é PixGG?

**PixGG** ([pixgg.com](https://pixgg.com)) é uma plataforma de doações e pagamentos para streamers e criadores de conteúdo. Permite gerar QR codes PIX e receber pagamentos em tempo real.

### Vantagens
- ✅ Gera QR codes PIX automaticamente
- ✅ API REST simples
- ✅ Notificações em tempo real via Pusher (websocket)
- ✅ Suporte a múltiplas moedas e criptomoedas
- ✅ Dashboard web para gerenciar pagamentos

---

## Configuração Inicial

### 1. Criar Conta no PixGG

1. Acesse [pixgg.com](https://pixgg.com)
2. Crie uma conta de streamer
3. Configure sua chave PIX no dashboard
4. Anote seu **Streamer ID** (encontrado na URL do seu perfil ou nas configurações)

### 2. Obter Credenciais

Você precisará de:
- **Streamer ID** (número único da sua conta)
- **Email** (usado para login)
- **Senha** (da sua conta PixGG)

### 3. Configurar Variáveis de Ambiente

Crie um arquivo `.env` na raiz do projeto:

```env
PIXGG_STREAMER_ID=82378
PIXGG_EMAIL=seu@email.com
PIXGG_PASSWORD=sua-senha-aqui
```

### 4. Instalar Dependências

```bash
npm install axios qrcode
```

---

## Endpoints Disponíveis

### Base URL
```
https://app.pixgg.com
```

### 1. **Login (Autenticação JWT)**

**Endpoint:** `POST /users/login`

**Payload:**
```json
{
  "name": "",
  "email": "seu@email.com",
  "password": "sua-senha"
}
```

**Headers:**
```javascript
{
  'Content-Type': 'application/json',
  'Origin': 'https://pixgg.com',
  'Referer': 'https://pixgg.com/',
  'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'
}
```

**Resposta:**
```json
{
  "username": "SeuNome",
  "authToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "needsToSetUsername": false,
  "needsAdditionalInfo": false,
  "apiKey": "17b330bb-572f-4c59-bc28-c5a22136cd2e"
}
```

> ⚠️ O `authToken` expira após ~1 hora. Implemente cache e renovação automática.

---

### 2. **Criar Pagamento PIX**

**Endpoint:** `POST /checkouts`

**Headers:**
```javascript
{
  'Content-Type': 'application/json',
  'Origin': 'https://pixgg.com',
  'Referer': 'https://pixgg.com/',
  'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36'
}
```

**Payload:**
```json
{
  "streamerId": 82378,
  "donatorNickname": "João Silva",
  "donatorMessage": "Pagamento VIP 30 dias",
  "donatorAmount": 24.90,
  "country": "Brazil",
  "cryptoCoin": null,
  "cryptoNetwork": "ETH",
  "fileId": null,
  "minimumDonateAmount": null,
  "songPixYoutubeUrl": "",
  "YouTubeVideoStart": 0,
  "YouTubeVideoEnd": 0,
  "youTubeVideoId": ""
}
```

**Resposta:**
```json
{
  "transactionId": 3315690,
  "pixUrl": "00020101021226830014BR.GOV.BCB.PIX2561qrcodespix...",
  "status": 5,
  "tokenAmount": null,
  "cryptoAddress": null,
  "paymentLink": null,
  "payment": 1
}
```

**Status:**
- `5` = Pendente (aguardando pagamento)
- `1` = Pago/Confirmado

---

### 3. **Verificar Pagamento**

**Endpoint:** `GET /checkouts/:transactionId`

**Headers:**
```javascript
{
  'Authorization': 'Bearer SEU_AUTH_TOKEN',
  'Content-Type': 'application/json',
  'Origin': 'https://pixgg.com',
  'Referer': 'https://pixgg.com/'
}
```

**Resposta:**
```json
{
  "transactionId": 3315690,
  "status": 1,
  "amount": 24.90,
  "donatorNickname": "João Silva",
  "createdAt": "2026-05-27T10:30:00Z"
}
```

> ⚠️ **Nota:** Este endpoint retorna 404 em alguns casos. A PixGG usa **Pusher** (websocket) para notificações em tempo real. Alternativa: verificar diferença de saldo antes/depois.

---

### 4. **Consultar Saldo**

**Endpoint:** `GET /BankAccounts/statistics`

**Headers:**
```javascript
{
  'Authorization': 'Bearer SEU_AUTH_TOKEN',
  'Content-Type': 'application/json',
  'Origin': 'https://pixgg.com',
  'Referer': 'https://pixgg.com/'
}
```

**Resposta:**
```json
{
  "available": 1.92,
  "avaliable": 1.92,
  "waitingAmount": 0.00,
  "isPocket": true
}
```

**Campos:**
- `available` / `avaliable`: Saldo disponível para saque
- `waitingAmount`: Saldo pendente (aguardando confirmação)
- `isPocket`: Se é conta pocket (carteira digital)

---

## Implementação Básica

### Módulo `pixgg.js`

```javascript
const axios = require('axios');

const BASE_URL = 'https://app.pixgg.com';

const CONFIG = {
  streamerId: parseInt(process.env.PIXGG_STREAMER_ID),
  email: process.env.PIXGG_EMAIL,
  password: process.env.PIXGG_PASSWORD,
};

let _authToken = null;
let _tokenExpiry = 0;

// ─── Login (obtém JWT) ────────────────────────────────────────────────────────
async function login() {
  if (_authToken && Date.now() < _tokenExpiry) return _authToken;

  const { data } = await axios.post(`${BASE_URL}/users/login`, {
    name: '',
    email: CONFIG.email,
    password: CONFIG.password,
  }, {
    headers: {
      'Content-Type': 'application/json',
      'Origin': 'https://pixgg.com',
      'Referer': 'https://pixgg.com/',
      'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36',
    },
  });

  _authToken = data.authToken;
  _tokenExpiry = Date.now() + 55 * 60 * 1000; // 55 minutos

  return _authToken;
}

// ─── Gerar Pagamento PIX ──────────────────────────────────────────────────────
async function generatePixPayment({ amount, donatorName, message }) {
  const payload = {
    streamerId: CONFIG.streamerId,
    donatorNickname: donatorName || 'Anônimo',
    donatorMessage: message || '',
    donatorAmount: amount,
    country: 'Brazil',
    cryptoCoin: null,
    cryptoNetwork: 'ETH',
    fileId: null,
    minimumDonateAmount: null,
    songPixYoutubeUrl: '',
    YouTubeVideoStart: 0,
    YouTubeVideoEnd: 0,
    youTubeVideoId: '',
  };

  const { data } = await axios.post(`${BASE_URL}/checkouts`, payload, {
    headers: {
      'Content-Type': 'application/json',
      'Origin': 'https://pixgg.com',
      'Referer': 'https://pixgg.com/',
      'User-Agent': 'Mozilla/5.0',
    },
  });

  return {
    pixUrl: data.pixUrl,
    transactionId: data.transactionId,
    status: data.status,
    paymentToken: String(data.transactionId),
  };
}

// ─── Verificar Pagamento ──────────────────────────────────────────────────────
async function checkPayment(transactionId) {
  const token = await login();

  const { data } = await axios.get(`${BASE_URL}/checkouts/${transactionId}`, {
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json',
      'Origin': 'https://pixgg.com',
      'Referer': 'https://pixgg.com/',
    },
  });

  const confirmed = [1, '1', 'paid', 'confirmed'].includes(data.status);

  return { confirmed, status: data.status, raw: data };
}

// ─── Consultar Saldo ──────────────────────────────────────────────────────────
async function getBalance() {
  const token = await login();

  const { data } = await axios.get(`${BASE_URL}/BankAccounts/statistics`, {
    headers: {
      'Authorization': `Bearer ${token}`,
      'Content-Type': 'application/json',
      'Origin': 'https://pixgg.com',
      'Referer': 'https://pixgg.com/',
    },
  });

  return {
    available: data.available ?? data.avaliable ?? 0,
    pending: data.waitingAmount ?? 0,
    total: (data.available ?? 0) + (data.waitingAmount ?? 0),
  };
}

module.exports = { login, generatePixPayment, checkPayment, getBalance };
```

---

## Fluxo de Pagamento Completo

### 1. Gerar Pagamento

```javascript
const { generatePixPayment } = require('./pixgg');

const pix = await generatePixPayment({
  amount: 24.90,
  donatorName: 'João Silva',
  message: 'VIP 30 dias',
});

console.log('QR Code PIX:', pix.pixUrl);
console.log('ID da Transação:', pix.transactionId);
```

### 2. Gerar QR Code (Imagem)

```javascript
const QRCode = require('qrcode');

// Gera buffer PNG
const qrBuffer = await QRCode.toBuffer(pix.pixUrl, {
  type: 'png',
  width: 300,
  margin: 2,
  color: { dark: '#000000', light: '#ffffff' },
});

// Envia no Discord
const attachment = new AttachmentBuilder(qrBuffer, { name: 'qrcode.png' });

await interaction.reply({
  content: 'Escaneie o QR Code para pagar:',
  files: [attachment],
});
```

### 3. Enviar Embed com QRCODE e Chave pix (cópiar/colar)

```javascript
const { EmbedBuilder } = require('discord.js');

const embed = new EmbedBuilder()
  .setTitle('💳 Pagamento PIX')
  .setDescription(`**Valor:** R$${pix.amount.toFixed(2)}`)
  .setImage('attachment://qrcode.png')
  .addFields({
    name: '📋 Copia e Cola',
    value: `\`\`\`${pix.pixUrl}\`\`\``,
  })
  .setFooter({ text: `ID: ${pix.transactionId} • Expira em 15 minutos` });

await interaction.reply({
  embeds: [embed],
  files: [attachment],
});
```

---

## Verificação de Pagamento

### Método 1: Polling (Verificação Periódica)

```javascript
const { checkPayment } = require('./pixgg');

async function waitForPayment(transactionId, maxAttempts = 30) {
  for (let i = 0; i < maxAttempts; i++) {
    try {
      const status = await checkPayment(transactionId);
      
      if (status.confirmed) {
        console.log('✅ Pagamento confirmado!');
        return true;
      }
      
      console.log(`⏳ Tentativa ${i + 1}/${maxAttempts} — Status: ${status.status}`);
      await new Promise(resolve => setTimeout(resolve, 10000)); // 10s
    } catch (err) {
      console.error('Erro ao verificar:', err.message);
    }
  }
  
  return false;
}

// Uso
const paid = await waitForPayment(pix.transactionId);
if (paid) {
  // Libera o VIP
}
```

### Método 3: Verificação por Diferença de Saldo

```javascript
const { getBalance } = require('./pixgg');

// Antes de gerar o pagamento
const balanceBefore = await getBalance();

// Gera o pagamento
const pix = await generatePixPayment({ amount: 24.90 });

// Aguarda 30 segundos
await new Promise(resolve => setTimeout(resolve, 30000));

// Verifica saldo novamente
const balanceAfter = await getBalance();

if (balanceAfter.available > balanceBefore.available) {
  console.log('✅ Pagamento detectado!');
}
```

---

## Consulta de Saldo

### Comando `/versaldo` (Owner-Only)

```javascript
const { SlashCommandBuilder } = require('discord.js');
const { getBalance } = require('./pixgg');

module.exports = {
  data: new SlashCommandBuilder()
    .setName('versaldo')
    .setDescription('Consulta o saldo da conta PixGG'),
  
  async execute(interaction) {
    // Verifica se é o dono do bot
    if (interaction.user.id !== 'SEU_USER_ID') {
      return interaction.reply({ content: '❌ Apenas o dono pode usar este comando.', ephemeral: true });
    }
    
    await interaction.deferReply({ ephemeral: true });
    
    try {
      const balance = await getBalance();
      
      await interaction.editReply({
        content:
          `💰 **Saldo PixGG**\n\n` +
          `💵 Disponível: **R$${balance.available.toFixed(2)}**\n` +
          `⏳ Pendente: **R$${balance.pending.toFixed(2)}**\n` +
          `📊 Total: **R$${balance.total.toFixed(2)}**`,
      });
    } catch (err) {
      await interaction.editReply({ content: `❌ Erro: ${err.message}` });
    }
  },
};
```

---

## Troubleshooting

### Erro 403 (Forbidden)

**Causa:** Headers incorretos ou faltando autenticação.

**Solução:**
- Verifique se está usando `Authorization: Bearer TOKEN` nos endpoints autenticados
- Adicione headers `Origin` e `Referer` corretos
- Certifique-se de que o token JWT não expirou

### Erro 404 no `/checkouts/:id`

**Causa:** Endpoint pode não estar disponível ou transação não existe.

**Solução:**
- Use verificação por diferença de saldo
- Implemente Pusher (websocket) para notificações em tempo real
- Aguarde alguns segundos antes de verificar (pagamentos levam 5-30s para confirmar)

### Erro 502 (Bad Gateway)

**Causa:** Servidor PixGG temporariamente indisponível.

**Solução:**
- Implemente retry com backoff exponencial
- Aguarde 60 segundos antes de tentar novamente
- Verifique status do serviço em [status.pixgg.com](https://status.pixgg.com) (se disponível)

### Token JWT Expira Rapidamente

**Solução:**
```javascript
let _authToken = null;
let _tokenExpiry = 0;

async function login() {
  // Cache por 55 minutos (token expira em 60)
  if (_authToken && Date.now() < _tokenExpiry) {
    return _authToken;
  }
  
  // Faz login novamente
  const { data } = await axios.post(...);
  _authToken = data.authToken;
  _tokenExpiry = Date.now() + 55 * 60 * 1000;
  
  return _authToken;
}
```

### QR Code Não Funciona

**Causa:** `pixUrl` pode estar vazio ou inválido.

**Solução:**
```javascript
if (!pix.pixUrl || pix.pixUrl.length < 50) {
  throw new Error('QR Code inválido');
}

// Valida formato BR Code
if (!pix.pixUrl.startsWith('00020101')) {
  throw new Error('Formato BR Code inválido');
}
```
---

## 📝 Licença

Este guia é fornecido "como está", sem garantias. Use por sua conta e risco.

---

**Criado com ❤️ por Kalebinho**
