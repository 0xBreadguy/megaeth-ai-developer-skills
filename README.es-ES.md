# Habilidad de Desarrollador de MegaETH para Agentes de IA

Una habilidad exhaustiva para agentes de codificación de IA (Claude Code, OpenClaw, Codex) para construir aplicaciones en tiempo real sobre el stack actual de MegaETH. Se mantiene consciente de las especificaciones para la depuración y el comportamiento histórico, pero trata el comportamiento actual de la era MegaEVM / REX5 como la línea base predeterminada.

## Descripción General

Esta habilidad proporciona a los agentes de IA un conocimiento profundo del ecosistema de desarrollo de MegaETH:

- **Transacciones**: `eth_sendRawTransactionSync` (EIP-7966) para el retorno síncrono de recibos de baja latencia sin necesidad de polling.
- **Patrones RPC**: Lotes de `eth_call` priorizando Multicall, keepalive de WebSocket, suscripciones a mini-bloques.
- **Almacenamiento**: Patrones de optimización para evitar costos costosos de SSTORE.
- **Modelo de Gas**: Costos específicos de MegaEVM y estrategias de estimación.
- **Depuración**: CLI de mega-evme para la reproducción de transacciones y el perfilado de gas.
- **Seguridad**: Consideraciones específicas de MegaETH y listas de verificación de auditoría.
- **MOSS**: Stack de flujo de trabajo de billetera MegaETH y claves delegadas orientado a desarrolladores y agentes de codificación.
- **USDm**: La stablecoin nativa de MegaETH y una primitiva central de pago/aplicación.
- **VRF / Aleatoriedad**: Integración del verificador de drand quicknet como parte del stack de desarrollo de MegaETH.
- **ERC-8004**: Patrones de agentes sin confianza y recursos de identidad/reputación de los que los desarrolladores deben estar conscientes.

## Instalación

### Instalación Rápida (skills.sh)

```bash
npx skills add 0xBreadguy/megaeth-ai-developer-skills
```

### Instalación Manual

```bash
git clone https://github.com/0xBreadguy/megaeth-ai-developer-skills
# Copiar al directorio de habilidades de tu agente
```

### OpenClaw / ClawdHub

```bash
clawdhub install megaeth-ai-developer-skills
```

## Estructura de la Habilidad

```
├── SKILL.md                  # Habilidad principal (decisiones de stack, procedimiento operativo)
└── skills/
    └── megaeth-developer/
        ├── SKILL.md
        └── references/
            ├── wallet-operations.md
            ├── frontend-patterns.md
            ├── rpc-methods.md
            ├── smart-contracts.md
            ├── storage-optimization.md
            ├── gas-model.md
            ├── testing.md
            ├── mega-evme.md
            ├── security.md
            ├── erc8004-trustless-agents.md
            ├── vrf-drand.md
            ├── usdm-stablecoin.md
            └── resources.md
```

## Uso

Una vez instalada, tu agente de IA utilizará automáticamente esta habilidad cuando preguntes sobre:

- Construcción de dApps en MegaETH.
- Envío y confirmación de transacciones.
- Desarrollo de contratos inteligentes con MegaEVM.
- Optimización de almacenamiento y costos de gas.
- Suscripciones de WebSocket en tiempo real.
- Depuración de transacciones fallidas.
- Reproducción o depuración local de transacciones de MegaETH con mega-evme.
- Comprender cuándo usar Foundry frente a mega-evme para el diagnóstico.

### Ejemplos de Prompts

```
"Configura una billetera para MegaETH"
"Envía 0.1 ETH en MegaETH"
"Intercambia USDM por ETH en MegaETH"
"Puentea ETH desde Ethereum a MegaETH"
"Configura una aplicación Next.js con conexión de billetera MegaETH"
"Despliega un contrato en MegaETH con Foundry"
"¿Por qué mi transacción está consumiendo tanto gas?"
"¿Cómo me suscribo a mini-bloques en tiempo real?"
"Optimiza este contrato para los costos de almacenamiento de MegaETH"
"Depura esta transacción fallida en MegaETH"
"Construye un flujo de lotería o revelación con drand VRF en MegaETH"
"¿Cómo debo integrar la aleatoriedad de forma segura en MegaETH?"
"Integra los flujos de trabajo de la billetera MOSS en una aplicación de MegaETH"
"Usa USDm en una aplicación o flujo de pago de MegaETH"
"Explica los patrones de agentes sin confianza ERC-8004 en MegaETH"
```

### ¿Qué archivo debe leer el agente?

- `testing.md` → pruebas generales, flujos de trabajo de Foundry, resolución de problemas comunes.
- `mega-evme.md` → reproducción local, análisis de trazas, depuración de MegaEVM consciente de la especificación.
- `smart-contracts.md` → restricciones de diseño de contratos, contratos del sistema, comportamiento de datos volátiles.

## Conceptos Clave

### Recibos de Transacciones Síncronos

MegaETH soporta `eth_sendRawTransactionSync` (EIP-7966), que permite el retorno síncrono de recibos de baja latencia en lugar de requerir un bucle de polling separado:

```typescript
const receipt = await client.request({
  method: 'eth_sendRawTransactionSync',
  params: [signedTx]
});
// El recibo se devuelve en el mismo flujo de RPC
```

### Conciencia de la Especificación (Spec Awareness)

El comportamiento de MegaETH sigue versiones de especificación, pero este repositorio trata el comportamiento documentado actual de la era MegaEVM / REX5 como la línea base predeterminada. Recurre a las advertencias específicas de actualizaciones solo al depurar comportamientos históricos, apuntar a estados de red antiguos o validar diferencias entre la implementación y la especificación.

### Costos de Almacenamiento

Los nuevos slots de almacenamiento son costosos (2M+ de gas). La habilidad enseña a los agentes a:
- Usar `RedBlackTreeLib` de Solady en lugar de mappings.
- Diseñar para la reutilización de slots.
- Considerar el almacenamiento off-chain para datos voluminosos.

### Modelo de Gas

MegaETH tiene una tarifa base estable de 0.001 gwei sin ajustes de EIP-1559. La habilidad enseña a los agentes a:
- Omitir estimaciones de gas innecesarias.
- Usar estimaciones remotas (los costos de MegaEVM difieren de la EVM estándar).
- Hardcodear los límites de gas para operaciones conocidas.

## Configuración de la Cadena

| Red | Chain ID | RPC | Explorador |
|---------|----------|-----|----------|
| Mainnet | 4326 | `https://mainnet.megaeth.com/rpc` | `https://mega.etherscan.io` |
| Testnet | 6343 | `https://carrot.megaeth.com/rpc` | `https://megaeth-testnet-v2.blockscout.com` |

## Divulgación Progresiva

La habilidad utiliza la divulgación progresiva: el archivo `SKILL.md` principal proporciona la guía central, y el agente lee archivos especializados solo cuando es necesario para tareas específicas. Esto mantiene el contexto eficiente mientras proporciona una experiencia profunda cuando se requiere.

## Alcance del Ecosistema

Este repositorio es intencionalmente opinionado respecto al stack central de desarrollo de MegaETH:
- Comportamiento de la plataforma/runtime de MegaETH.
- MOSS / MOSS CLI / MOSS Skills para flujos de trabajo de billetera y claves delegadas.
- MOSS Skills como guía para desarrolladores/agentes para integrar MOSS en aplicaciones.
- USDm como una primitiva central de aplicación/pago.
- drand VRF como una primitiva central de aleatoriedad.
- ERC-8004 como un recurso importante de agentes sin confianza que los desarrolladores deben comprender.

Para habilidades de MegaETH específicas de protocolo y aplicación más allá de este stack central, consulta [Awesome MegaETH AI](https://github.com/megaeth-labs/awesome-megaeth-ai).

## Fuentes de Contenido

Esta habilidad incorpora las mejores prácticas de:

- [Documentación Oficial de MegaETH](https://docs.megaeth.com)
- [Especificación de MegaEVM](https://github.com/megaeth-labs/mega-evm)
- [EIP-7966 (eth_sendRawTransactionSync)](https://ethereum-magicians.org/t/eip-7966-eth-sendrawtransactionsync-method/24640)
- [MOSS Skills](https://github.com/megaeth-labs/moss-skills)
- [MegaETH Wallet CLI](https://github.com/megaeth-labs/wallet-cli)
- [Awesome MegaETH AI](https://github.com/megaeth-labs/awesome-megaeth-ai)
- Guía técnica del equipo de MegaETH

## Contribución

¡Las contribuciones son bienvenidas! Por favor, asegúrate de que las actualizaciones reflejen las mejores prácticas actuales del ecosistema MegaETH.

1. Haz un fork del repositorio.
2. Crea una rama para la funcionalidad (feature branch).
3. Realiza tus cambios.
4. Envía un pull request.

## Licencia

MIT
