# Arquitetura Técnica do FubicaDroid

> **Documento de Arquitetura de Software Embarcado e Edge Computing**  
> *Padrão: Open Architecture, Closed Core — Princípios, padrões de projeto e decisões estruturais (ADRs).*

---

## 1. Visão Geral da Arquitetura Edge-to-Cloud

O **FubicaDroid** foi projetado para operar como um **nó de processamento de borda (*Edge Computing*) de missão crítica**. Ele transforma um smartphone Android padrão em um computador de bordo veicular de alto desempenho, eliminando a dependência de telas proprietárias de montadoras ou rastreadores telemáticos invasivos.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       CAMADA VEICULAR (IN-VEHICLE)                          │
│                                                                             │
│   ECU Stellantis           Adaptador OBD-II              Central Multimídia │
│  (Barramento CAN)           (CHX ELM327)               (Citroën ConnectPlay)│
│        │                          ▲                              ▲          │
│        │ Linha K / CAN            │ RFCOMM Bluetooth             │ USB      │
│        ▼                          ▼                              ▼          │
│ ┌───────────────┐        ┌──────────────────┐          ┌──────────────────┐ │
│ │ Porta OBD-II  ├───────►│  SPP Serial Link ├─────────►│  Android Auto     │ │
│ │ (Pino 16: 12V)│        │ (11-bit 500kbaud)│          │  (CarAppService) │ │
│ └───────────────┘        └────────┬─────────┘          └────────▲─────────┘ │
└───────────────────────────────────┼─────────────────────────────┼───────────┘
                                    │                             │
┌───────────────────────────────────┼─────────────────────────────┼───────────┐
│                                   ▼                             │           │
│                   FUBICADROID EDGE ENGINE (SMARTPHONE)          │           │
│                                                                 │           │
│  ┌───────────────────────────────────────────────────────────┐  │           │
│  │ Anti-Kill Background Service (ForegroundService)          │  │           │
│  │ • connectedDevice + location (Android 14/15/16)           │  │           │
│  │ • Hardware Watchdog & Event-Driven Reconnect              │  │           │
│  └─────────────────────────────┬─────────────────────────────┘  │           │
│                                │                                │           │
│  ┌─────────────────────────────▼─────────────────────────────┐  │           │
│  │ Driver Serial Resiliente (Sanitização de Buffer & Parser) │  │           │
│  │ • Drenagem de Stream pré-comando • Tratamento de Timings  │  │           │
│  └─────────────────────────────┬─────────────────────────────┘  │           │
│                                │ StateFlow                      │           │
│  ┌─────────────────────────────▼─────────────────────────────┐  │           │
│  │ Motor de Fusão de Dados & Física Speed-Density            │  │           │
│  │ • Termodinâmica GSE Firefly • Fused Location Provider GPS │  │           │
│  └──────┬──────────────────────┬──────────────────────┬──────┘  │           │
│         │                      │                      │         │           │
│         ▼                      ▼                      ▼         │           │
│  ┌──────────────┐       ┌──────────────┐       ┌────────────────┴────────┐  │
│  │ Cockpit HUD  │       │ SQLite Local │       │ Projeção Veicular       │  │
│  │ Jetpack      │       │ Buffer de    │       │ Android Auto Templates  │  │
│  │ Compose 60FPS│       │ Sessão       │       │ CarAppService Oficial   │  │
│  └──────────────┘       └──────┬───────┘       └─────────────────────────┘  │
└────────────────────────────────┼────────────────────────────────────────────┘
                                 │ HTTPS / API
┌────────────────────────────────┼────────────────────────────────────────────┐
│                                ▼                                            │
│                      CAMADA CLOUD & ANALYTICS                               │
│                                                                             │
│               ┌──────────────────────────────────────┐                      │
│               │ Pipeline Car Monitor Analytics       │                      │
│               │ • Ingestão Assíncrona                │                      │
│               │ • Dashboard de Eficiência Energética │                      │
│               │ • Auditoria de Frotas & Telemetria   │                      │
│               └──────────────────────────────────────┘                      │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Resiliência em Segundo Plano (Anti-Kill em Android 14/15/16 & Xiaomi HyperOS)

### 2.1. O Problema das Camadas de Otimização dos Fabricantes
Sistemas operacionais Android modernos — com destaque para o **Xiaomi HyperOS / MIUI**, Samsung OneUI e versões stock do Android 14+ — impõem políticas extremamente severas de gerenciamento de energia:
- Eliminação agressiva de processos em background via `lmkd` (*Low Memory Killer Daemon*).
- Restrição de inicialização de serviços de localização em background.
- Fechamento arbitrário de conexões Bluetooth ativas por inatividade de toque na tela.

### 2.2. A Solução Arquitetural do FubicaDroid
Para garantir rastreamento ininterrupto durante viagens de centenas de quilômetros com a tela do celular apagada no bolso do motorista:

1. **Classificação Rigorosa do Foreground Service:**
   O serviço principal opera em conformidade estrita com o manifesto do Android 14+:
   - Tipos de Serviço Declarados: `android:foregroundServiceType="connectedDevice|location"`.
   - Inicialização escalonada: O serviço inicializa como `connectedDevice` ao detectar o pareamento Bluetooth e somente promove o binding de `location` (GPS) após confirmar o enlace serial ativo com a ECU.

2. **Gerenciamento Inteligente de WakeLocks:**
   - Utilização de `PARTIAL_WAKE_LOCK` cirúrgico durante a transmissão serial ativa.
   - Liberação imediata de lock ao detectar corte de ignição do veículo, impedindo que o aplicativo consuma a bateria do celular quando o carro for estacionado.

3. **Arquitetura Orientada a Eventos de Hardware (Zero Polling Idle):**
   - Eliminação de loops infinitos de busca Bluetooth quando fora do veículo.
   - O despertar do serviço é orquestrado por um `BroadcastReceiver` registrado para eventos do sistema:
     - `BluetoothDevice.ACTION_ACL_CONNECTED`
     - `BluetoothDevice.ACTION_ACL_DISCONNECTED`
     - `BluetoothHeadset.ACTION_CONNECTION_STATE_CHANGED`
   - O app só aloca recursos de CPU se o endereço MAC conectado corresponder ao adaptador telemático ou à central multimídia cadastrada.

---

## 3. Driver de Comunicação Serial Resiliente (Camada OBD-II / ELM327)

### 3.1. Isolamento de Conexão RFCOMM
A camada de comunicação opera sobre um socket **Bluetooth Classic RFCOMM** dedicado, ancorado no UUID padrão serial:
```text
UUID: 00001101-0000-1000-8000-00805F9B34FB (SerialPortServiceClass_UUID)
```

### 3.2. Sanitização de Buffer e Proteção contra Clones CHX
Adaptadores OBD-II de baixo custo (*clones ELM327 v1.5 / CHX*) sofrem de problemas crônicos de transceptor serial:
- Buffers de UART minúsculos (64 a 128 bytes).
- Acúmulo de caracteres residuais gerando respostas concatenadas corrompidas.
- Travamento do transceptor CAN em caso de bombardeio de requisições rápidas.

**Padrões de Engenharia Aplicados no FubicaDroid:**
- **Drenagem Ativa Pré-Comando:** O fluxo de entrada (`InputStream`) é drenado por completo antes da emissão de qualquer comando AT ou consulta de PID, garantindo integridade de frame.
- **Cadência Adaptativa de Enfileiramento:** O loop de transmissão serial implementa uma cadência de `120 ms` entre quadros completos, estabilizando a taxa em ~3 a 4 Hz sem sobreaquecer o chip ELM.
- **Boot Delay de Hardware:** Pausa determinística de estabilização pós-`ATZ` (reset de hardware) permitindo o término do ciclo de inicialização do microcontrolador.
- **Watchdog de Timeouts:** Mecanismo de tolerância a 25 falhas consecutivas antes de declarar a ECU inativa, evitando desconexões acidentais em semáforos com sistema *Start&Stop*.

---

## 4. Integração Oficial Android Auto (`androidx.car.app`)

Diferente de soluções legadas que dependiam de *root* ou espelhamento não-oficial de tela (AA Mirror, Screen2Auto), o FubicaDroid implementa a API oficial **Android for Cars App Library**:

```
┌────────────────────────────────────────────────────────┐
│                   ANDROID AUTO HOST                    │
│           (Google Play Services Car Library)           │
└───────────────────────────┬────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────┐
│             FubicaCarAppService (Oficial)              │
│  Declaração de Categoria: POI / Internet of Things     │
└───────────────────────────┬────────────────────────────┘
                            │
                            ▼
┌────────────────────────────────────────────────────────┐
│                  FubicaCarSession                      │
│  Gerencia ciclo de vida da tela veicular               │
└───────────────────────────┬────────────────────────────┘
                            │ StateFlow Subscriptions
                            ▼
┌────────────────────────────────────────────────────────┐
│                CarDashboardScreen                      │
│  Renderização via Templates Automotivos de Baixa       │
│  Distração (PaneTemplate / NavigationTemplate)         │
│  • Cards com Extended ASCII CP437 HUD                  │
│  • Barras de Bargraph dinâmicas                        │
│  • Métricas essenciais: RPM, MAP, Temp, Consumo, Marcha│
└────────────────────────────────────────────────────────┘
```

### Benefícios da Arquitetura:
1. **Conformidade de Segurança:** Interface projetada para minimizar a distração do motorista, respeitando os requisitos de contraste e legibilidade veicular do Google.
2. **Independência de Resolução:** Ajuste dinâmico automático tanto para telas horizontais (ex.: Citroën ConnectPlay de 10 polegadas) quanto para formatos verticais.
3. **Persistência Visual:** A interface no veículo continua atualizando mesmo se o celular estiver bloqueado ou com outra aplicação aberta em primeiro plano.

---

## 5. Renderização de Alta Performance (Jetpack Compose & HUD Noturno)

Para a interface no smartphone, o FubicaDroid adota uma arquitetura reativa moderna:

- **Desacoplamento de Threads:** A thread de I/O Bluetooth deposita os dados telemétricos processados em um `MutableStateFlow<TelemetrySnapshot>`. A thread de UI consome esse fluxo de forma não-bloqueante.
- **Canvas Direto sem Recomposição Excessiva:** Ponteiros analógicos, tacômetro circular e barras de carga são desenhados diretamente na camada de `Canvas` do Jetpack Compose, garantindo taxa de atualização constante de **60 FPS** com baixo consumo de GPU.
- **Modo HUD Espelho Noturno:**
  Inversão geométrica horizontal da matriz de renderização (`scale(scaleX = -1f, scaleY = 1f)`) e esquema de cores em alto contraste (fundo preto absoluto `#000000` e tipografia verde/azul de alta intensidade). Permite posicionar o celular sobre o painel do veículo para refletir os mostradores diretamente no para-brisa sem reflexos parasitas.

---

## 6. Persistência de Dados e Sincronização Cloud

1. **Armazenamento em Borda (Edge Storage):**
   - Banco relacional **SQLite** local operando como buffer de sessão.
   - Amostragem otimizada: gravação consolidada a cada 5 segundos para equilibrar granularidade de traçado GPS com eficiência de escrita em memória flash.
2. **Filtro de Sessões Inválidas:**
   - Algoritmo que audita `MAX(RPM)` e `MAX(Speed)`. Sessões onde o veículo não se movimentou ou o motor não deu partida são descartadas antes da persistência em disco.
3. **Upload Assíncrono com Fila de Retentativa:**
   - Os arquivos de viagem exportados entram em uma fila persistente (`pending_uploads`).
   - Caso o veículo trafegue por áreas de sombra de cobertura de dados móveis (3G/4G/5G), os dados são mantidos em cache local e transmitidos automaticamente assim que uma conexão estável for restabelecida.
