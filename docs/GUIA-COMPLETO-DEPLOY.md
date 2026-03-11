# Guia Completo de Deploy - Biblioteca Universitaria

## Visao Geral da Arquitetura

```
┌─────────────────────────────────────────────────────────────────┐
│                        INTERNET                                 │
└───────────────────────────┬─────────────────────────────────────┘
                            │
              ┌─────────────┴─────────────┐
              │                           │
        ┌─────▼─────┐               ┌─────▼─────┐
        │  VERCEL   │    HTTPS      │  RENDER   │
        │ (Frontend)│ ◄───────────► │ (Backend) │
        │  React    │               │  Express  │
        └─────┬─────┘               └─────┬─────┘
              │                           │
        [frontend/]                 [backend/]
        [build/]                    [Node.js]
        [SPA React]                 [Prisma]
                                          │
                                    ┌─────▼─────┐
                                    │ PostgreSQL│
                                    │ (Database)│
                                    └───────────┘
```

---

## Parte 1: Configuracoes Atuais do Projeto

### Arquivo: `vercel.json` (na raiz)

```json
{
  "version": 2,
  "buildCommand": "cd frontend && npm install && npm run build",
  "outputDirectory": "frontend/build",
  "cleanUrls": false,
  "rewrites": [
    { "source": "/(.*)", "destination": "/index.html" }
  ],
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        { "key": "X-Content-Type-Options", "value": "nosniff" },
        { "key": "X-Frame-Options", "value": "SAMEORIGIN" },
        { "key": "X-XSS-Protection", "value": "1; mode=block" },
        { "key": "Referrer-Policy", "value": "strict-origin-when-cross-origin" },
        { "key": "Permissions-Policy", "value": "camera=(), microphone=(), geolocation=()" }
      ]
    }
  ]
}
```

**O que cada configuracao faz:**

| Propriedade | Funcao |
|-------------|--------|
| `buildCommand` | Navega para `/frontend`, instala dependencias e builda |
| `outputDirectory` | Indica onde esta o resultado do build |
| `rewrites` | Redireciona todas as rotas para `index.html` (necessario para SPA React) |
| `headers` | Headers de seguranca aplicados a todas as respostas |

### Frontend: `package.json`

```json
{
  "engines": { "node": "22.x" },
  "scripts": {
    "build": "react-scripts build"
  }
}
```

---

## Parte 2: Passo a Passo - Deploy do Frontend na Vercel

### Passo 1: Acessar a Vercel

1. Acesse [vercel.com](https://vercel.com)
2. Faca login com sua conta **GitHub**
3. Clique em **"Add New Project"**

### Passo 2: Importar o Repositorio

1. Selecione o repositorio **Estudok410/Biblioteca-Universitaria**
2. Aguarde a Vercel detectar o projeto

### Passo 3: Configurar o Build

No painel da Vercel, configure:

| Campo | Valor |
|-------|-------|
| **Framework Preset** | `Create React App` ou `Other` |
| **Root Directory** | `.` (deixe vazio - usa a raiz) |
| **Build Command** | Deixe vazio (usa o `vercel.json`) |
| **Output Directory** | Deixe vazio (usa o `vercel.json`) |
| **Node.js Version** | `22.x` |

### Passo 4: Configurar Variaveis de Ambiente

Expanda **"Environment Variables"** e adicione:

| Nome | Valor | Descricao |
|------|-------|-----------|
| `REACT_APP_API_URL` | `https://sua-api.onrender.com` | URL do backend no Render |

**Importante:** Selecione todos os ambientes (Production, Preview, Development).

### Passo 5: Deploy

1. Clique em **"Deploy"**
2. Aguarde o build (1-3 minutos)
3. Ao finalizar, voce recebera uma URL como: `https://biblioteca-universitaria.vercel.app`

---

## Parte 3: Configuracao do CORS no Backend (Render)

### O que e CORS?

**CORS (Cross-Origin Resource Sharing)** e uma protecao dos navegadores que **bloqueia requisicoes entre dominios diferentes**.

```
┌─────────────────────────────────────────────────────────────────┐
│ Sem CORS configurado:                                           │
│                                                                 │
│  [Frontend Vercel]  ──── requisicao ────►  [Backend Render]    │
│  biblioteca.vercel.app                      api.onrender.com    │
│                                                                 │
│        ▲                                                        │
│        │                                                        │
│   NAVEGADOR BLOQUEIA!                                           │
│   "Origem diferente nao permitida"                              │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ Com CORS configurado:                                           │
│                                                                 │
│  [Frontend Vercel]  ──── requisicao ────►  [Backend Render]    │
│  biblioteca.vercel.app                      api.onrender.com    │
│                                                    │            │
│                                     "Essa origem esta permitida"│
│                                              CORS_ORIGIN        │
│                                                                 │
│        ◄───────────── resposta ─────────────────────┘           │
│   NAVEGADOR PERMITE!                                            │
└─────────────────────────────────────────────────────────────────┘
```

### Situacao Atual do Backend

O codigo atual usa `app.use(cors())` **sem restricoes**, o que significa que **aceita qualquer origem**. Isso funciona, mas **nao e recomendado para producao**.

### Opcao A: Manter como esta (Mais Simples)

Se o backend esta assim:
```javascript
app.use(cors());
```

**Nao precisa fazer nada no Render.** O backend ja aceita requisicoes de qualquer origem, incluindo a Vercel.

**Ponto negativo:** Qualquer site pode fazer requisicoes para a API.

### Opcao B: Restringir por Seguranca (Recomendado)

Para restringir apenas ao frontend:

**1. Modifique o arquivo `backend/src/index.js`:**

```javascript
require("dotenv").config();
const express = require("express");
const cors = require("cors");

const app = express();

// Configuracao segura do CORS
const corsOptions = {
  origin: process.env.CORS_ORIGIN || 'http://localhost:3000',
  credentials: true
};
app.use(cors(corsOptions));

// ... resto do codigo
```

**2. Configure a variavel no Render:**

No dashboard do Render:
1. Selecione o servico backend
2. Va em **Environment** no menu lateral
3. Adicione ou edite:

| Variavel | Valor |
|----------|-------|
| `CORS_ORIGIN` | `https://biblioteca-universitaria.vercel.app` |

4. Clique em **Save Changes**
5. O Render vai reiniciar automaticamente

### Opcao C: Multiplas Origens (Desenvolvimento + Producao)

Se quiser permitir tanto localhost quanto producao:

```javascript
// backend/src/index.js
const allowedOrigins = (process.env.CORS_ORIGIN || 'http://localhost:3000').split(',');

app.use(cors({
  origin: function(origin, callback) {
    // Permitir requisicoes sem origin (ex: Postman, apps mobile)
    if (!origin) return callback(null, true);
    
    if (allowedOrigins.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error('Bloqueado pelo CORS'));
    }
  },
  credentials: true
}));
```

No Render, configure:
```
CORS_ORIGIN=http://localhost:3000,https://biblioteca-universitaria.vercel.app
```

---

## Parte 4: Conectar Frontend ao Backend

### Criar arquivo de configuracao da API

Crie `frontend/src/services/api.js`:

```javascript
const API_URL = process.env.REACT_APP_API_URL || 'http://localhost:3001';

export async function fetchAPI(endpoint, options = {}) {
  const response = await fetch(`${API_URL}${endpoint}`, {
    headers: {
      'Content-Type': 'application/json',
      ...options.headers,
    },
    ...options,
  });
  
  if (!response.ok) {
    throw new Error(`Erro na API: ${response.status}`);
  }
  
  return response.json();
}

export default API_URL;
```

### Usar nos componentes

Exemplo para o Catalogo:

```javascript
import { useState, useEffect } from 'react';
import { fetchAPI } from '../services/api';

function Catalogo() {
  const [livros, setLivros] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchAPI('/api/livros')
      .then(data => setLivros(data))
      .catch(err => console.error(err))
      .finally(() => setLoading(false));
  }, []);

  // ... resto do componente
}
```

---

## Parte 5: Checklist Final

### Antes do Deploy

- [ ] Repositorio atualizado no GitHub
- [ ] `vercel.json` configurado na raiz
- [ ] `package.json` do frontend tem `"engines": { "node": "22.x" }`

### Deploy do Frontend (Vercel)

- [ ] Projeto importado na Vercel
- [ ] Node.js 22.x selecionado
- [ ] `REACT_APP_API_URL` configurada com URL do Render
- [ ] Deploy realizado com sucesso

### Deploy do Backend (Render) - Se necessario

- [ ] `CORS_ORIGIN` configurada com URL da Vercel (se usar Opcao B ou C)
- [ ] Servico reiniciado apos mudancas

### Testes Pos-Deploy

- [ ] Abrir o site da Vercel no navegador
- [ ] Verificar se a pagina carrega
- [ ] Testar login (se implementado)
- [ ] Verificar Console do navegador (F12) para erros de CORS

---

## Parte 6: Troubleshooting

### Erro: "Build failed"

**Causa provavel:** Versao do Node.js incorreta

**Solucao:**
1. Na Vercel, va em Settings > General
2. Altere Node.js Version para `22.x`
3. Re-deploy

### Erro: "CORS policy: No 'Access-Control-Allow-Origin'"

**Causa:** Backend nao permite a origem do frontend

**Solucao:**
1. Verifique se `CORS_ORIGIN` no Render inclui a URL da Vercel
2. Certifique-se de que a URL esta **exatamente igual** (com ou sem `/` no final)
3. Reinicie o servico no Render

### Erro: "Failed to fetch" ou "Network Error"

**Causa provavel:** URL da API incorreta

**Solucao:**
1. Verifique `REACT_APP_API_URL` na Vercel
2. Certifique-se de que a URL nao tem barra no final
3. Teste a URL diretamente no navegador

### Pagina em branco apos deploy

**Causa:** Rotas do React nao funcionando

**Solucao:** Ja resolvido pelo `rewrites` no `vercel.json`:
```json
"rewrites": [{ "source": "/(.*)", "destination": "/index.html" }]
```

---

## Resumo Rapido

| Plataforma | Pasta | Variaveis Necessarias |
|------------|-------|----------------------|
| **Vercel** | `frontend/` | `REACT_APP_API_URL` |
| **Render** | `backend/` | `CORS_ORIGIN` (se restringir) |

---

## Observacoes Importantes

1. **O `vercel.json` na raiz NAO afeta o backend no Render** - sao plataformas independentes
2. **O backend atualmente aceita qualquer origem** - funciona, mas considere restringir por seguranca
3. **Variaveis `REACT_APP_*` sao incorporadas no build** - mudancas requerem novo deploy
4. **O Render reinicia automaticamente** apos mudancas em variaveis de ambiente
