
<html lang="pt-BR">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Exploração de assetKey em S3 - Análise Técnica</title>
  <style>
    body { font-family: Arial, sans-serif; line-height: 1.6; padding: 2rem; max-width: 900px; margin: auto; background: #f9f9f9; color: #333; }
    h1, h2, h3 { color: #222; font-family: Arial, sans-serif; }
    pre { background: #eee; padding: 1rem; overflow-x: auto; border-radius: 5px; }
    code { font-family: monospace; color: #c7254e; }
    table { border-collapse: collapse; width: 100%; margin: 1rem 0; }
    th, td { border: 1px solid #ccc; padding: 8px; text-align: left; }
    th { background: #eee; }
    .summary { background: #e0f7fa; padding: 1rem; border-left: 5px solid #0097a7; margin-bottom: 1rem; font-family: Arial, sans-serif; }
  </style>
</head>
<body>
  <h1>Exploração de assetKey com Traversal Profundo em S3</h1>

  <div class="summary">
    <p>Este artigo explora uma vulnerabilidade de path traversal persistente usando o parâmetro <code>assetKey</code>, refletido em URLs pré-assinadas da Amazon S3 e automaticamente buscado pelo frontend, permitindo exfiltração de dados e DoS persistente.</p>
  </div>

  <h2>Introdução</h2>
  <p>Durante um processo de análise de segurança web, descobri uma vulnerabilidade interessante envolvendo uploads de imagens e parâmetros ofuscados. Neste artigo, compartilho minha linha de raciocínio, ferramentas utilizadas e como a exploração técnica se desenvolveu até a comprovação de impacto.</p>

  <h2>Linha de Raciocínio</h2>

  <h3>Passo 1: Observando o comportamento do WAF</h3>
  <p>O primeiro passo foi entender como o WAF e o backend reagiam a entradas maliciosas. Comecei testando se os inputs eram decodificados. Após isso, utilizei técnicas de obfuscação e payloads XSS em parâmetros encontrados com a ferramenta <strong>ParamSpider</strong>. Ao perceber que o site fazia <em>decode</em>, testei <code>URL encoded</code> duplo, poliglotas e notei que o WAF não bloqueava esses encodes.</p>

  <h3>Passo 2: Ferramentas de apoio e enumeração</h3>
  <p>Com a confirmação de que o sistema aceitava encode profundo, usei o <strong>ffuf</strong> para descobrir parâmetros ocultos, além do <strong>Burp Suite Intruder</strong> para testar meus payloads de forma mais focada. Utilizei também o <strong>Dalfox</strong> para automatizar parte dos testes em busca de XSS.</p>

  <h3>Passo 3: Testes em upload de imagem</h3>
  <p>Ao testar manualmente os parâmetros, encontrei um campo de upload de imagens. Troquei o nome da imagem por payloads <code>duplamente codificados</code> e, apesar de aceitos, eles não acionavam scripts XSS. Percebi então que o backend estava armazenando o nome, mas aplicava alguma sanitização parcial.</p>

  <h3>Passo 4: Explorando path traversal</h3>
  <p>Decidi mudar a abordagem e testar payloads de <strong>Path Traversal</strong>. Submeti arquivos com nomes como <code>../../config.json</code> duplamente codificados, que passaram pelo filtro. Notei que a aplicação gerava uma URL S3 <em>pré-assinada</em> automaticamente para cada upload, persistente e inapagável — um possível <strong>DoS</strong>.</p>

  <h3>Passo 5: Entendendo o formato dos buckets</h3>
  <p>Pesquisei sobre a estrutura dos buckets S3. Descobri que muitos diretórios não seguiam a estrutura <code>../../</code>, mas sim <code>/config.json</code> direto. Ao interceptar as requisições das imagens, recebi erros do tipo <em>bad request</em> que revelavam nome, região e outros metadados do bucket.</p>

  <h3>Passo 6: Enumeração com wordlists e ferramentas</h3>
  <p>Com as informações coletadas, usei o <strong>s3scanner</strong> e o <strong>ffuf</strong> com uma wordlist focada em arquivos comuns como <code>.env</code>, <code>config.json</code>, <code>.git/config</code>. Em um dos diretórios, consegui uma resposta positiva com dados sensíveis.</p>

  <h2>Detalhamento Técnico</h2>

  <h3>Resumo da vulnerabilidade:</h3>
  <ul>
    <li>AssetKeys persistem mesmo após remoção do UI</li>
    <li>Requisições automáticas são feitas pelo frontend</li>
    <li>Permite leitura de arquivos internos do bucket</li>
    <li>Exploração silenciosa sem interação do usuário</li>
    <li>Potencial para DoS, LFI</li>
  </ul>

  <h3>Payload usado no upload:</h3>
  <pre><code>%2525252E%2525252E%2525252Fconfig.json</code></pre>

  <p>Que decodifica três vezes para:</p>
  <pre><code>../../config.json</code></pre>

  <h3>Requisição automática gerada:</h3>
  <pre><code>GET /50545022/%2525252E%2525252E%2525252Fconfig.json?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=...
Host: [REMOVIDO].s3.[REGIÃO].amazonaws.com
Referer: https://[DOMÍNIO-REDACTED]/mypetprofile/pet</code></pre>

  <h2>Tabela de Impacto</h2>
  <table>
    <thead>
      <tr><th>Vetor de Ataque</th><th>Resultado</th></tr>
    </thead>
    <tbody>
      <tr><td>Traversal Armazenado</td><td>Acesso a objetos como <code>../../config.json</code></td></tr>
      <tr><td>Reflexão Automática</td><td>assetKey embutido automaticamente na URL</td></tr>
      <tr><td>URLs Pré-assinadas</td><td>Geração de links confiáveis para payloads</td></tr>
      <tr><td>Sem interação do usuário</td><td>Browser carrega automaticamente</td></tr>
      <tr><td>DoS</td><td>Arquivo inválido permanece acessível</td></tr>
      <tr><td>XSS Latente</td><td>Se assetKey for refletido no HTML</td></tr>
      <tr><td>Enumeração de Bucket</td><td>Baseada nos códigos de resposta</td></tr>
      <tr><td>Bypass de WAF</td><td>Payloads codificados profundamente não são bloqueados</td></tr>
    </tbody>
  </table>

  <footer>
    <p><em>Escrito com o objetivo de compartilhar conhecimento técnico. Nenhuma informação sensível ou confidencial foi exposta.</em></p>
  </footer>
</body>
</html>
