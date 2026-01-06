# InfraStatusChecker — Monitoramento simples (API .NET 6 + HTML/CSS/JS + SVG)

**InfraStatusChecker** é um painel leve de monitoramento que:
- consulta **hosts e serviços** (internos e externos),
- consolida **contadores** e **alertas** via **API REST** (.NET 6),
- renderiza um **diagrama SVG interativo** (com IDs mapeáveis) e **pinta/pisca** elementos afetados,
- toca um **bip** quando surgirem **novos alertas**.

> Feito para **rodar no IIS** (Windows Server) e ser acessado por navegador sem dependências pesadas.

---

## ✨ Destaques

- **Back-end**: ASP.NET Core 6 (minimal APIs), endpoint `/api/status`.
- **Front-end**: HTML estático + CSS + JavaScript puro.
- **SVG mapeável**: cada `<g class="djs-visual" id="...">` pode ser associado a um serviço/host para realce contextual.
- **Alertas audíveis**: notificação sonora (customizável) quando os alertas mudam.
- **Zero build no front**: basta copiar os arquivos para o IIS.
- **Sem impacto em sites legados** (.NET Framework) — app pool separado e módulo AspNetCoreModuleV2 isolado.

---

## 🗂️ Estrutura do repositório

```
InfraStatusChecker/
├─ api/
│  └─ InfraStatusChecker/           # Projeto .NET 6 (API)
│     ├─ Program.cs
│     ├─ Services/StatusService.cs
│     ├─ Models.cs
│     ├─ appsettings.json           # Configuração de targets, timeouts, etc.
│     └─ web.config                 # Hosting no IIS (OutOfProcess)
└─ web/
   ├─ DIAGRAMA.html                 # Página principal
   ├─ static/
   │  ├─ styles.css                 # Estilos do painel
   │  └─ app.js                     # Lógica: fetch, contadores, SVG highlight, áudio
   └─ audio/
      └─ alert.mp3                  # Som de alerta (pode trocar)
```

---

## ✅ Requisitos

- Windows Server (2012 R2/2016/2019/2022) com **IIS**.
- **.NET 6 Hosting Bundle** instalado (para rodar apps ASP.NET Core no IIS).
- Permissões de leitura nas pastas de publicação (`C:\inetpub\wwwroot\...`).

---

## 🚀 Quickstart (publicação no IIS)

### 1) Publicar API (.NET 6)

```bash
git clone <seu-repo>.git
cd InfraStatusChecker/api/InfraStatusChecker

dotnet publish -c Release -o C:\inetpub\wwwroot\api
```

> O `web.config` incluso já está pronto para **OutOfProcess** com `AspNetCoreModuleV2`.

### 2) Copiar o front para o IIS

Copie a pasta `web` para a raiz do site (ex.: `C:\inetpub\wwwroot\`), ou publique onde preferir:

```
C:\inetpub\wwwroot\
   ├─ api\        # API publicada no passo anterior
   └─ DIAGRAMA.html
      static\
      audio\
```

### 3) Configurar no IIS

- Em **Default Web Site**, crie um **aplicativo** chamado `api` apontando para `C:\inetpub\wwwroot\api`.
- App Pool: modo integrado; .NET CLR **Sem código gerido**.
- Reinicie o IIS:
```powershell
iisreset /restart
```

### 4) Testar

- `http://SEU_SERVIDOR/api/` → `{ "ok": true, "service": "InfraStatusChecker" }`
- `http://SEU_SERVIDOR/api/status` → JSON com `contadores`, `resultados`, `alertas`.
- `http://SEU_SERVIDOR/DIAGRAMA.html` → painel com contadores, faixa de alertas e diagrama.

---

## ⚙️ Configuração

### API_URL do front
`web/static/app.js`:
```js
// Produção via IIS (app "api" sob Default Web Site):
const API_URL = "/api/api/status";

// Local (self-host em 6005):
// const API_URL = "http://127.0.0.1:6005/api/status";
```

### Mapeamento nome → IDs do SVG
Também no html ajustar os mapeamentos:
```js
const MAP_NAME = {
  "TESTE": ["11"],
  "TESTE": ["19"],
  "TESTE": ["9"],
  "TESTE": ["23"]
};
```

> Garanta que seu **SVG** tenha IDs nos grupos que você quer iluminar (ex.: `<g class="djs-visual" id="11">…</g>`).  
> Recomendo inserir o **SVG inline** dentro do `DIAGRAMA.html` para facilitar o acesso ao DOM.

### Estilos e animações
`web/static/styles.css` contém as classes de estado e animação de piscar.

### Áudio de alerta
`web/audio/alert.mp3` pode ser substituído por qualquer som curto. No `app.js`, ajuste o volume:
```js
alertAudio.volume = 1.0;  // 0.0 a 1.0
```

---

## 🧠 Como funciona

1. **API** (`/api/status`)
   - Lê a lista de alvos de `appsettings.json`.
   - Para cada alvo:
     - Tipo **H** → faz GET e mede latência; **OK** se `2xx`.
     - Tipo **S** → faz teste ping e mede latência.
   - Produz:
     - `contadores`: `servidoresOk`, `servidoresAlerta`, `servicosOk`, `servicosAlerta`.
     - `resultados`: itens com `status`, `nome`, `endereco`, `latenciaMs`, `erro`.
     - `alertas`: mensagens curtas legíveis.

2. **Front**
   - Faz polling a cada **30s** (configurável).
   - Atualiza contadores, faixa de alertas, toca áudio ao detectar **mudança** nos alertas.
   - Pinta/pisca IDs mapeados no **SVG** conforme o status.

---

## 🛡️ Segurança

- Se exposto fora da rede, **restrinja** por firewall e/ou autentique o acesso ao painel.
- Avalie **CORS** se front e API estiverem em origens diferentes.
- Para HTTPS externos, instale a cadeia de certificados correspondente.

---

## 🧰 Troubleshooting

| Sintoma | Causa provável | Como resolver |
|---|---|---|
| `HTTP 500` no `/api` | Hosting Bundle ausente | Instale **.NET 6 Hosting Bundle** e reinicie o IIS |
| `/api/status` lento/vazio | Timeout curto ou URL inválida | Revise `appsettings.json` e conectividade |
| Front com `ERR_CONNECTION_REFUSED` | API fora do ar ou `API_URL` incorreta | Verifique o app `api` no IIS e o valor da constante |
| SVG não realça | IDs inexistentes ou `MAP_NAME` sem cobertura | Garanta `<g id="...">` e revise o mapeamento |
| Áudio não toca | Política do navegador | O som toca quando a lista de alertas **muda**; ajuste a lógica/volume |

---

## 📜 Licença

Este projeto usa a licença **MIT** (permissiva).  
Você pode **usar, copiar, modificar e distribuir** livremente, inclusive para uso comercial, desde que **mantenha o aviso de copyright e a licença**.

- Veja o arquivo [`LICENSE`](./LICENSE) neste repositório.
- Se preferir um modelo ainda mais “uso livre”/domínio público, considere **The Unlicense** ou **CC0** (troque o conteúdo do arquivo de licença).

---

## 📌 Créditos

Projeto pensado para monitoramento interno com diagrama operacional, priorizando simplicidade, baixo acoplamento e fácil manutenção por equipes de infraestrutura. Principal diferencia é poder montar o mapa do ambiente e saber de fato o que determinada indisponibilidade afeta.
