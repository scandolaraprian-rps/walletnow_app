# Documentação Técnica de Engenharia: Resiliência e Mecanismo de Retry no `StellarRpcClient`

---

## 1. Visão Geral e Contexto Arquitetural

No ecossistema de aplicações descentralizadas e Web3, a estabilidade e a resiliência na comunicação com os nós RPC (Remote Procedure Call) da rede Stellar/Soroban são críticas para garantir uma experiência de usuário (UX) fluida. Instabilidades transitórias de rede, picos de carga em nós validadores ou falhas intermitentes de infraestrutura (como erros HTTP `500 Internal Server Error` ou timeouts de conexão) não devem resultar imediatamente em falhas na interface ou quebras de transação.

Para mitigar esses problemas, introduzimos o **`StellarRpcClient`** no pacote `@walletnow/sdk`, implementando uma arquitetura robusta baseada em:
* **Mecanismo de *Retry* com Backoff Exponencial**: Tentativas automáticas inteligentes para erros recuperáveis.
* **Fast-Fail para Erros Client-Side (`4xx`)**: Otimização de performance evitando re-tentativas desnecessárias em requisições inválidas ou recursos inexistentes.
* **Integração Unificada via Fachada (`WalletNowClient`)**: Exposição transparente do cliente RPC através de `wn.rpc`.
* **Tratamento Padronizado de Erros (`NetworkError`)**: Mapeamento de falhas persistentes para contratos de erro consistentes do `@walletnow/contracts`.

---

## 2. Detalhamento das Mudanças Realizadas

### 2.1. Criação do `StellarRpcClient` em `packages/sdk`
O novo componente foi desenvolvido para encapsular o protocolo de comunicação HTTP/JSON-RPC com a rede Stellar. Ele introduz parâmetros configuráveis de resiliência:
* `maxRetries`: Número máximo de tentativas subsequentes em caso de falhas transitórias.
* `retryDelayMs`: Intervalo base para o cálculo do backoff exponencial.

### 2.2. Política de Falhas e Recuperação
O cliente adota uma estratégia distinta com base no tipo de resposta HTTP ou exceção de rede:
* **Erros do Servidor (`500+`) e Timeouts (`TimeoutError`)**: Considerados falhas transitórias. O sistema intercepta o erro, aplica o atraso configurado e dispara uma nova tentativa até o limite de `maxRetries`.
* **Erros do Cliente (`4xx`, ex: `404 Not Found`, `400 Bad Request`)**: Classificados como falhas definitivas. O mecanismo de *retry* é abortado imediatamente (*fast-fail*), propagando o erro original sem gastar recursos de rede adicionais.
* **Esgotamento de Tentativas**: Caso todas as tentativas falhem, o SDK lança uma instância de **`NetworkError`** (proveniente de `@walletnow/contracts`), sinalizando o esgotamento da política de resiliência.

### 2.3. Integração com a Fachada `WalletNowClient`
A fachada unificada (`WalletNowClient`) foi atualizada para inicializar e expor o cliente RPC internamente através da propriedade `wn.rpc`. Isso permite que desenvolvedores utilizem recursos avançados de rede sem precisar instanciar módulos de baixo nível manualmente.

### 2.4. Exportação de Tipos Canônicos
Todos os tipos, interfaces de configuração e contratos de resposta do módulo RPC foram exportados publicamente em `@walletnow/sdk`, garantindo tipagem estricta (`TypeScript`) para os consumidores do SDK.

---

## 3. Validação por Testes Unitários

A suíte de testes implementada cobre exaustivamente os cenários de resiliência, garantindo o cumprimento estrito dos critérios de aceite:

```typescript
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { StellarRpcClient } from '../rpc/StellarRpcClient.js';
import { WalletNowClient } from '../WalletNowClient.js';
import { NetworkError } from '@walletnow/contracts';

describe('@walletnow/sdk — StellarRpcClient Resilience & Retry Mechanism', () => {
  let fetchSpy: any;

  beforeEach(() => {
    vi.restoreAllMocks();
  });

  afterEach(() => {
    vi.restoreAllMocks();
  });

  it('deve tentar novamente quando a chamada RPC falhar com erro 500 e retornar com sucesso na segunda tentativa', async () => {
    let callCount = 0;
    fetchSpy = vi.spyOn(globalThis, 'fetch').mockImplementation((() => {
      callCount++;
      if (callCount === 1) {
        return Promise.resolve({
          ok: false,
          status: 500,
          statusText: 'Internal Server Error',
          text: () => Promise.resolve('Soroban RPC node returned 500 Internal Error'),
        } as Response);
      }

      return Promise.resolve({
        ok: true,
        status: 200,
        statusText: 'OK',
        json: () =>
          Promise.resolve({
            jsonrpc: '2.0',
            id: 1,
            result: {
              status: 'healthy',
              latestLedger: 998877,
            },
          }),
      } as Response);
    }) as any);

    const rpc = new StellarRpcClient({
      network: 'testnet',
      retry: {
        maxRetries: 3,
        retryDelayMs: 0,
      },
    });

    const result = await rpc.getHealth();

    // Critérios de Aceite:
    // 1. Provar que o SDK fez exatas duas requisições
    expect(fetchSpy).toHaveBeenCalledTimes(2);

    // 2. Retornar os dados do sucesso da segunda tentativa sem expor o erro ao usuário
    expect(result).toEqual({
      status: 'healthy',
      latestLedger: 998877,
    });
  });

  it('deve tentar novamente quando a chamada falhar por Timeout e retornar dados da segunda tentativa', async () => {
    let callCount = 0;
    fetchSpy = vi.spyOn(globalThis, 'fetch').mockImplementation((() => {
      callCount++;
      if (callCount === 1) {
        const timeoutError = new Error('Connection timed out');
        timeoutError.name = 'TimeoutError';
        return Promise.reject(timeoutError);
      }

      return Promise.resolve({
        ok: true,
        status: 200,
        statusText: 'OK',
        json: () =>
          Promise.resolve({
            jsonrpc: '2.0',
            id: 1,
            result: {
              sequence: '12345',
            },
          }),
      } as Response);
    }) as any);

    const rpc = new StellarRpcClient({
      network: 'testnet',
      retry: {
        maxRetries: 3,
        retryDelayMs: 0,
      },
    });

    const result = await rpc.getAccount('GBTEST...');

    // Critérios de Aceite:
    expect(fetchSpy).toHaveBeenCalledTimes(2);
    expect(result).toEqual({ sequence: '12345' });
  });

  it('deve funcionar transparentemente via WalletNowClient.init()', async () => {
    let callCount = 0;
    fetchSpy = vi.spyOn(globalThis, 'fetch').mockImplementation((() => {
      callCount++;
      if (callCount === 1) {
        return Promise.resolve({
          ok: false,
          status: 500,
          statusText: 'Internal Server Error',
          text: () => Promise.resolve('Internal server error on ledger retrieval'),
        } as Response);
      }

      return Promise.resolve({
        ok: true,
        status: 200,
        statusText: 'OK',
        json: () =>
          Promise.resolve({
            jsonrpc: '2.0',
            id: 1,
            result: {
              ledgerNumber: 887766,
            },
          }),
      } as Response);
    }) as any);

    const client = await WalletNowClient.init({
      network: 'testnet',
      rpc: {
        retry: {
          maxRetries: 3,
          retryDelayMs: 0,
        },
      },
    });

    const result = await client.rpc.getLatestLedger();

    expect(fetchSpy).toHaveBeenCalledTimes(2);
    expect(result).toEqual({ ledgerNumber: 887766 });
  });

  it('não deve retentar chamadas em erros 4xx de cliente', async () => {
    fetchSpy = vi.spyOn(globalThis, 'fetch').mockImplementation((() => {
      return Promise.resolve({
        ok: false,
        status: 404,
        statusText: 'Not Found',
        text: () => Promise.resolve('Account not found'),
      } as Response);
    }) as any);

    const rpc = new StellarRpcClient({
      network: 'testnet',
      retry: {
        maxRetries: 3,
        retryDelayMs: 0,
      },
    });

    await expect(rpc.call('getAccount', { address: 'UNKNOWN' })).rejects.toThrow('RPC HTTP Error 404');
    expect(fetchSpy).toHaveBeenCalledTimes(1);
  });

  it('deve lançar NetworkError ao esgotar todas as tentativas configuradas', async () => {
    fetchSpy = vi.spyOn(globalThis, 'fetch').mockImplementation((() => {
      return Promise.resolve({
        ok: false,
        status: 500,
        statusText: 'Internal Server Error',
        text: () => Promise.resolve('Persistent RPC failure'),
      } as Response);
    }) as any);

    const rpc = new StellarRpcClient({
      network: 'testnet',
      retry: {
        maxRetries: 2,
        retryDelayMs: 0,
      },
    });

    await expect(rpc.getHealth()).rejects.toThrow(NetworkError);
    expect(fetchSpy).toHaveBeenCalledTimes(2);
  });
});