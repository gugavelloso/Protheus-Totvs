---

# Protheus REST API Validation Utilities

## Objetivo

Oferecer uma estrutura robusta e reutilizável em **TLPP** para validar requisições HTTP recebidas por APIs REST no **TOTVS Protheus**, desacoplando a lógica de validação de cabeçalhos e corpo da lógica de negócios principal do endpoint.

---

## Visão Geral

O conjunto de classes padroniza a validação de duas partes críticas de uma requisição REST:

* **Cabeçalho (Header)**: valida empresa e filial através do header definido em `MV_XCHKHDR`.
* **Corpo (Body)**: garante que o corpo da requisição não esteja vazio e contenha JSON válido.

A orquestração é feita pela classe **`validationRequestApi`**, que utiliza:

* **`HttpHeaderTools`** → validação do cabeçalho.
* **`HttpBodyUtils`** → validação do corpo.

---

## Parâmetros

* **MV_XCHKHDR**: parâmetro de ambiente no Protheus que define o nome do header a ser validado.
* **Tenant**: valor esperado no formato `"empresa,filial"`.

---

## Propósito / Regras

* Promover **código limpo**: regras de validação separadas da lógica de negócios.
* Garantir **segurança**: rejeitar requisições malformadas.
* Padronizar **tratamento de erros** (HTTP 400 / HTTP 422).

---

## Formato Esperado

### Header

```http
Tenant: 01,01
```

### Body

```json
{
  "campo1": "valor",
  "campo2": 123
}
```

---

## Estrutura das Classes

### 1. `custom.httpvalidationrequestapi.tlpp`

**Classe:** `validationRequestApi`
**Função:** Orquestrar a validação completa.

**Principais métodos:**

* `new()`: inicializa instâncias de `HttpHeaderTools` e `HttpBodyUtils`.
* `validate()`: executa validação sequencial → cabeçalho e corpo.

---

### 2. `custom.httpheaderutils.tlpp`

**Classe:** `HttpHeaderTools`
**Função:** Validar cabeçalho `Tenant`.

**Principais métodos:**

* `getHeaderTenant()`:

  * Obtém o nome do header a partir de **MV_XCHKHDR**.
  * Verifica presença do header na requisição.
  * Garante formato `empresa,filial`.
  * Valida existência via `FwFilExist()`.
  * Em caso de falha retorna erro **400** (Bad Request) ou **422** (Unprocessable Entity).

---

### 3. `custom.httpbodyutils.tlpp`

**Classe:** `HttpBodyUtils`
**Função:** Validar corpo JSON.

**Principais métodos:**

* `checkBodyRequest()`:

  * Verifica se o body não está vazio.
  * Tenta parsing direto em JSON.
  * Se falhar → tenta reparse após decode UTF-8.
  * Se persistir → retorna **400 Bad Request** com mensagem do parser.
  * Em caso de sucesso → JSON é salvo em `oJsonBody`.

---

## Tratamento de Erros

* Erros são encapsulados no objeto **oRest**, com:

  * **Status HTTP (400 / 422)**
  * **Mensagem descritiva**

O endpoint apenas verifica se `validate()` retornou `.T.` ou `.F.` para decidir prosseguir.

---

## Fluxos de Uso

### Exemplo no endpoint:

```tlpp
User Function ApiExample()
    Local oValidation := validationRequestApi():new()
    
    If !oValidation:validate()
        Return // interrupção automática
    EndIf
    
    // A partir daqui, lógica de negócio usando:
    // oValidation:oHeader:getCompany()
    // oValidation:oHeader:getBranch()
    // oValidation:oBody:oJsonBody
Return
```

---

## Boas Práticas

* Definir **MV_XCHKHDR** em ambiente Protheus para padronização.
* Utilizar **validate()** como primeira instrução em cada endpoint REST.
* Centralizar todas as regras de validação em `HttpHeaderTools` e `HttpBodyUtils`.
* Tratar erros no nível das utilitárias, mantendo o endpoint **enxuto e seguro**.

---

## Diagrama de Fluxo (Mermaid)

Abaixo segue um diagrama em **Mermaid** que representa o fluxo de validação: header → body → negócio.

```mermaid
flowchart TD
    A[Receber requisição HTTP] --> B[Instanciar validationRequestApi]
    B --> C{Validar Header}
    C -- Header ausente/formato inválido --> C1[Responder 400 Bad Request]
    C -- Header inválido (empresa/filial não existe) --> C2[Responder 422 Unprocessable Entity]
    C -- Header válido --> D{Validar Body}
    D -- Body ausente --> D1[Responder 400 Bad Request]
    D -- JSON inválido --> D2[Responder 400 Bad Request (parse error)]
    D -- JSON válido --> E[Lógica de negócio (endpoint) usa: oHeader, oBody]
    E --> F[Processamento e resposta ao cliente]
    C1 --> G[Fim]
    C2 --> G
    D1 --> G
    D2 --> G
    F --> G
```

---

## Observações Finais

* Este conjunto de utilitárias torna os endpoints mais testáveis e seguros ao garantir que requisições inválidas sejam barradas cedo.
* Ajustes podem ser feitos para suportar formatos alternativos de header ou regras adicionais de validação.



