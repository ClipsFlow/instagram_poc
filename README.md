# Instagram OAuth & Reels Upload --- Proof of Concept

POC de autenticação OAuth 2.0 e publicação automatizada de Reels via
Meta Graph API.

------------------------------------------------------------------------

## Objetivo

Validar a viabilidade técnica de:

-   Autenticar usuários via OAuth 2.0 usando Meta / Instagram Graph API
-   Obter permissões para publicação de conteúdo
-   Criar containers de mídia para Reels
-   Publicar vídeos programaticamente no perfil do usuário
-   Entender o fluxo de processamento, erros e restrições da API

------------------------------------------------------------------------

## Resultado da validação

| Critério                           | Status        | Observação                                   |
|-----------------------------------|---------------|-----------------------------------------------|
| OAuth 2.0 funciona                | ✅ GO         | Authorization Code Flow                      |
| Obtenção de access_token          | ✅ GO         | Via `/oauth/access_token`                    |
| Criação de container de mídia     | ✅ GO         | Depende de vídeo válido e acessível          |
| Publicação via media_publish      | ✅ GO         | Só funciona após container processado        |
| Escopos necessários               | ✅ GO         | `instagram_basic`, `instagram_content_publish` |
| Processamento assíncrono          | ⚠️ Atenção    | Container precisa estar FINISHED   

------------------------------------------------------------------------

## Visão geral do fluxo

``` mermaid
sequenceDiagram
    participant U as Usuário
    participant B as Backend (POC)
    participant M as Meta OAuth
    participant G as Instagram Graph API

    U->>B: Inicia login com Instagram
    B->>M: Redireciona para autorização
    M->>U: Tela de consentimento
    U->>M: Autoriza permissões
    M->>B: Retorna authorization_code
    B->>M: Troca code por access_token
    M->>B: Retorna access_token
    B->>G: Usa token para criar container de mídia
    G->>B: Retorna creation_id
    B->>G: Consulta status do container
    G->>B: status_code (PROCESSING / FINISHED)
    B->>G: Publica mídia
    G->>B: Resultado final
```

------------------------------------------------------------------------

## Fluxo OAuth 2.0 (Authorization Code)

``` mermaid
flowchart TD
    A[Usuário clica em conectar Instagram] --> B[Redireciona para Meta OAuth]
    B --> C[Usuário autoriza app]
    C --> D[Meta retorna authorization_code]
    D --> E[Backend troca code por access_token]
    E --> F[Token salvo para chamadas à API]
```

### URL de autorização

GET https://www.facebook.com/v19.0/dialog/oauth

Parâmetros principais:

-   client_id
-   redirect_uri
-   scope=instagram_basic,instagram_content_publish
-   response_type=code

------------------------------------------------------------------------

## Fluxo de Upload de Reels

``` mermaid
sequenceDiagram
    participant B as Backend
    participant G as Instagram Graph API

    B->>G: POST /{ig_user_id}/media
    Note right of G: Cria container de vídeo
    G->>B: Retorna creation_id

    loop Aguardar processamento
        B->>G: GET /{creation_id}?fields=status_code
        G->>B: PROCESSING / FINISHED
    end

    B->>G: POST /{ig_user_id}/media_publish
    G->>B: Post publicado
```

------------------------------------------------------------------------

## Fluxo detalhado

``` mermaid
flowchart TD
    A[Obter access_token] --> B[Criar container de mídia]
    B --> C{Container criado?}
    C -->|Não| X[Erro: mídia inválida ou URL inacessível]
    C -->|Sim| D[Aguardar processamento]

    D --> E{Status = FINISHED?}
    E -->|Não| D
    E -->|Sim| F[Publicar com media_publish]

    F --> G{Sucesso?}
    G -->|Sim| H[Reel publicado]
    G -->|Não| I[Erro de permissão / formato]
```

------------------------------------------------------------------------

## Endpoints utilizados

### OAuth

-   GET /dialog/oauth
-   GET /oauth/access_token

### Upload / Publicação

-   POST /{ig_user_id}/media
-   GET /{creation_id}?fields=status_code
-   POST /{ig_user_id}/media_publish

------------------------------------------------------------------------

## Requisitos do vídeo

-   Arquivo .mp4
-   URL pública direta
-   Preferencialmente formato vertical (9:16)
-   Codec H.264 + áudio AAC recomendado

------------------------------------------------------------------------

## Possíveis falhas e causas comuns

  -----------------------------------------------------------------------
  Erro                Causa provável
  ------------------- ---------------------------------------------------
  status_code = ERROR URL de vídeo inválida, vídeo inacessível ou formato
                      inválido

  creation_id ausente Falha ao criar container (token ou mídia inválida)

  Falha no publish    Container ainda em processamento

  403 / Permissões    Token inválido ou falta de escopos
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## Conclusão

A POC demonstra que o fluxo de OAuth + upload de Reels é funcional
usando a Graph API, porém depende fortemente de:

-   URL pública e válida do vídeo
-   Processamento assíncrono do container
-   Permissões corretas (instagram_content_publish)
