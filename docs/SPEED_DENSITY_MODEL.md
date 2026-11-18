# Modelo Computacional Speed-Density (Estimativa Mássica Sem Sensor MAF)

> **Documento Conceitual de Engenharia Mecânica & Software Embarcado**  
> *Aplicação: Motores Aspirados Flex da Família Stellantis GSE Firefly (1.0 3-Cilindros / 1.3 4-Cilindros)*  
> *Classificação: Open Architecture, Closed Core — As constantes calibradas e tabelas de mapeamento são proprietárias.*

---

## 1. O Desafio Técnico: Ausência de Sensor MAF Físico

Em motores automotivos modernos de cilindrada reduzida (*downsizing* e alta eficiência termodinâmica), como a família **Stellantis GSE Firefly** presente em veículos como Citroën C3, Fiat Argo, Mobi, Cronos e Nova Strada, as montadoras optam rotineiramente por **eliminar o sensor MAF (*Mass Air Flow*) físico**.

### Motivações de Engenharia da Montadora
1. **Redução de Custo e Complexidade:** Eliminação do sensor de fio quente ou filme aquecido no duto de admissão.
2. **Minimização de Restrição Aerodinâmica:** Ausência de obstáculo físico ao fluxo de ar na admissão, maximizando o coeficiente de enchimento dos cilindros.
3. **Robustez:** Imunidade a contaminações por vapores de óleo do respiro do cárter (*blow-by* / PCV).

### A Consequência para Scanners e Telemetrias Genéricas
Scanners OBD-II convencionais de mercado (Torque Pro, Car Scanner, scanners universais de oficina) dependem da leitura direta do PID padrão SAE `0110` (Taxa de fluxo de ar MAF em $g/s$). Em veículos da plataforma GSE Firefly:
- A ECU retorna `NO DATA` ou valor nulo para o PID `0110`.
- Scanners genéricos exibem consumo zerado, taxas de vazão inconsistentes ou recorrem a aproximações grosseiras baseadas em médias estáticas de velocidade, incapazes de refletir a dinâmica real de aceleração, carga e freio motor (*cut-off*).

---

## 2. A Solução Implementada no FubicaDroid

O **FubicaDroid** embarca um **Motor de Inferência Física Speed-Density** executado em tempo real na camada de borda (*Edge Computing* no smartphone Android). O modelo reconstrói continuamente o fluxo mássico de ar ($\dot{m}_{air}$) através dos princípios fundamentais da termodinâmica dos gases e da geometria do motor.

```
                  ┌──────────────────────┐
                  │ Sensores da ECU      │
                  │ • MAP (kPa)          │
                  │ • IAT (°C -> Kelvin) │
                  │ • RPM (Rot/min)      │
                  └──────────┬───────────┘
                             │
                             ▼
                  ┌──────────────────────┐
                  │ Densidade do Ar      │
                  │ ρ = (P · M) / (R · T)│
                  └──────────┬───────────┘
                             │
                             ▼
┌──────────────────────┐  ┌──┴───────────────────┐
│ Deslocamento Vd (L)  ├─►│ Volume Teórico       │
│ Ciclo 4-Tempos       │  │ Aspirado por Segundo │
└──────────────────────┘  └──────────┬───────────┘
                                     │
┌──────────────────────┐             ▼
│ Superfície VE 3D     │  ┌──────────────────────┐
│ VE = f(RPM, MAP)     ├─►│ Fluxo Mássico de Ar  │
│ (Closed-Core)        │  │ ṁ_air (g/s)          │
└──────────────────────┘  └──────────┬───────────┘
                                     │
┌──────────────────────┐             ▼
│ Estequiometria Flex  │  ┌──────────────────────┐  ┌──────────────────────┐
│ AFR / Densidade Comb.├─►│ Vazão Horária Comb.  ├─►│ • Consumo (km/L)     │
│ Target Lambda        │  │ ṁ_fuel (L/h)         │  │ • Potência Est. (cv) │
└──────────────────────┘  └──────────────────────┘  └──────────────────────┘
```

---

## 3. Formulação Termodinâmica e Mecânica

### 3.1. Densidade do Ar Admitido ($\rho_{air}$)
A densidade do ar presente no coletor de admissão é derivada diretamente da Equação de Estado dos Gases Ideais:

\[
P \cdot V = \frac{m}{M_{air}} \cdot R \cdot T \implies \rho_{air} = \frac{P \cdot M_{air}}{R \cdot T}
\]

Onde:
- $P$: Pressão Absoluta do Coletor (**MAP**), obtida via PID OBD-II `010B` (convertida para $Pa$);
- $M_{air}$: Massa molar aparente do ar atmosférico seco ($\approx 0{,}0289644 \text{ kg/mol}$);
- $R$: Constante universal dos gases ($8{,}314462 \text{ J/(mol}\cdot\text{K)}$);
- $T$: Temperatura Absoluta do Ar Admitido (**IAT**), obtida via PID OBD-II `010F` (convertida de Celsius para Kelvin: $T = T_{IAT} + 273{,}15$).

Simplificando para unidades de engenharia automotiva com constante específica do ar seco ($R_{específico} \approx 287{,}058 \text{ J/(kg}\cdot\text{K)}$):

\[
\rho_{air} = \frac{P_{MAP} \times 1000}{287{,}058 \times (T_{IAT} + 273{,}15)} \quad [\text{kg/m}^3]
\]

---

### 3.2. Volume Teórico de Ar Varrido ($\dot{V}_{theor}$)
Em um motor a combustão de quatro tempos, cada cilindro completa um ciclo de admissão a cada duas rotações do virabrequim:

\[
\dot{V}_{theor} = \frac{RPM}{2 \times 60} \cdot V_d \quad [\text{m}^3/\text{s}]
\]

Onde:
- $RPM$: Rotação do motor (PID `010C`);
- $V_d$: Cilindrada volumétrica total do motor (ex.: $0{,}999 \times 10^{-3} \text{ m}^3$ para o Firefly 1.0 GSE).

---

### 3.3. Eficiência Volumétrica ($VE$) e Massa Real Ingerida
O volume geométrico teórico não é preenchido em sua totalidade devido à dinâmica dos dutos de admissão, perda de carga na borboleta de aceleração (TPS), perfil dos cames do comando de válvulas e contra-pressão de escape.

Essa relação adimensional é a **Eficiência Volumétrica ($VE$)**:

\[
VE = \frac{V_{real}}{V_{theor}} = f(RPM, MAP)
\]

Combinando a densidade do ar, o volume teórico varrido e a eficiência volumétrica dinâmica, o **Fluxo Mássico de Ar ($\dot{m}_{air}$)** admitido por segundo é calculado como:

\[
\dot{m}_{air} = \dot{V}_{theor} \cdot \rho_{air} \cdot VE(RPM, MAP) \quad [\text{g/s}]
\]

---

### 3.4. Dedução da Vazão de Combustível ($\dot{m}_{fuel}$) e Vazão Horária ($L/h$)
A partir da massa de ar admitida e da relação estequiométrica ar/combustível alvo ($AFR_{target}$), calcula-se a massa de combustível injetada:

\[
\dot{m}_{fuel} = \frac{\dot{m}_{air}}{AFR_{target} \cdot \lambda_{trim}} \quad [\text{g/s}]
\]

A conversão para **Vazão Horária Volumétrica ($Q_{L/h}$)** considera a densidade do combustível líquido ($\rho_{fuel}$):

\[
Q_{L/h} = \frac{\dot{m}_{fuel} \times 3600}{\rho_{fuel} \times 1000} \quad [\text{L/h}]
\]

---

### 3.5. Métricas Resultantes no FubicaDroid

1. **Consumo Instantâneo ($km/L$):**
   \[
   Consumo_{km/L} = \frac{Velocidade_{km/h}}{Q_{L/h}}
   \]
   *(Com proteção contra divisão por zero e comutação para $L/h$ quando $Velocidade < 3 \text{ km/h}$).*

2. **Detecção de Cut-Off (Freio Motor / Injeção Zero):**
   Quando a borboleta está fechada ($TPS \le limiar$), rotação acima da marcha lenta ($RPM > 1.200$) e vácuo alto ($MAP < limiar$), o motor entra em desaceleração com corte integral de injeção ($\dot{m}_{fuel} = 0 \implies Q_{L/h} = 0{,}0$).

3. **Estimativa de Potência Mecânica Instantânea ($P_{cv}$):**
   Derivada da taxa de energia química liberada pelo combustível multiplicada pelo rendimento térmico indicado ($\eta_{th}$) do motor Firefly:
   \[
   P_{cv} \approx \frac{\dot{m}_{fuel} \cdot PCI_{fuel} \cdot \eta_{th}(RPM, MAP)}{735{,}49875}
   \]

---

## 4. Salvaguardas de Propriedade Intelectual (Closed-Core)

> [!IMPORTANT]
> **O que é Aberto (Showcase):**
> As equações físicas clássicas da mecânica dos fluidos e termodinâmica descritas acima.
> 
> **O que é Proprietário e Restrito (Closed Core):**
> - **Superfície 3D de $VE$:** As matrizes de interpolação não-linear de eficiência volumétrica calibradas especificamente para o cabeçote de 2 válvulas por cilindro e comando único com variador de fase do motor GSE Firefly.
> - **Compensação Dinâmica Flex:** O algoritmo proprietário de inferência do teor de etanol e ajuste de $AFR_{target}$ sem sensor físico de combustível.
> - **Filtro de Amortecimento de Transientes:** O algoritmo de suavização exponencial para suprimir picos induzidos por ressonância no coletor sem adicionar atraso perceptual nos mostradores do HUD.
