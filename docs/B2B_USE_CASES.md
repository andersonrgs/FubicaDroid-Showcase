# Casos de Uso B2B & Viabilidade Comercial

> **Documento de Aplicação Comercial e Oportunidades de Mercado**  
> *Ecossistema: FubicaDroid (Edge Engine) + Car Monitor (Analytics Platform)*  
> *Classificação: Open Architecture, Closed Core — Proposta de Valor B2B.*

---

## 1. Visão Geral da Oportunidade

A telemetria automotiva corporativa tradicional sofre de três gargalos críticos:
1. **Custo de Hardware Elevado:** Rastreadores telemáticos dedicados (OBD dongles com modem 4G integrado) custam entre R$ 400 e R$ 1.200 por veículo, além de taxas mensais de conectividade de dados (chips M2M).
2. **Instalações Invasivas:** Rastreadores cabeados demandam corte de fiação e violam a garantia de fábrica de veículos novos da frota.
3. **Métricas Superficiais:** Rastreadores de baixo custo limitam-se a rastreamento GPS e ignição ligada/desligada, sem acesso à saúde termodinâmica da ECU ou telemetria em tempo real.

O ecossistema **FubicaDroid + Car Monitor** redefine esse paradigma transformando o smartphone corporativo que o motorista já utiliza (Android corporativo) em um **computador de telemetria avançada de ponta a ponta**, utilizando adaptadores OBD-II de baixo custo e zero invasão.

---

## 2. Aplicação 1: Gestão Inteligente de Frotas Leves Stellantis

### 2.1. O Cenário
A plataforma **Stellantis** domina as frotas comerciais urbanas no Brasil e na América Latina:
- **Veículos Típicos:** Fiat Strada, Fiat Fiorino, Fiat Mobi, Fiat Argo, Citroën C3 e Peugeot 208.
- **Motorizações:** Famílias GSE Firefly 1.0 (3 cilindros) e 1.3 (4 cilindros).

### 2.2. Dores Resolvidas pelo FubicaDroid

| Problema na Frota | Impacto Financeiro | Solução FubicaDroid |
| :--- | :--- | :--- |
| **Abuso de Rotação (Sobre-giro)** | Desgaste prematuro de bronzinas, correias e consumo excessivo. | Monitoramento contínuo com alarmes instantâneos para eventos de $RPM > 5.500$ ou reduções bruscas de marcha. |
| **Operação em Motor Frio** | Desgaste de cilindros por diluição de combustível no óleo. | Rastreamento da curva de aquecimento do líquido de arrefecimento ($ECT < 75^\circ\text{C}$). Alerta de condução agressiva com motor frio. |
| **Consumo Real de Combustível** | Fraude em abastecimentos e desvios de rota. | Auditoria independente baseada no modelo Speed-Density ($L/h$ e $km/L$), sem depender do marcador analógico do tanque. |
| **Falha Súbita de Bateria/Alternador** | Veículo parado em rota de entrega gerando custos de reboque e atrasos. | Registro da queda de tensão na partida (*cranking voltage sag*) e monitoramento contínuo da tensão do alternador sob carga ($13{,}8\text{V} - 14{,}4\text{V}$). |

### 2.3. Retorno sobre o Investimento (ROI Estimado para Frota de 50 Veículos)
- **Economia em Hardware:** Redução de ~80% no investimento inicial em comparação com hardwares telemáticos proprietários.
- **Eficiência de Combustível:** Redução comprovada de 7% a 14% no gasto com combustível decorrente do acompanhamento de eventos de condução agressiva e uso otimizado do freio motor (*cut-off*).

---

## 3. Aplicação 2: Triagem Rápida na Recepção de Oficinas Mecânicas & Concessionárias

### 3.1. O Cenário
Na recepção de uma oficina mecânica ou centro automotivo, a fase de *checklist de entrada* frequentemente depende de inspeções visuais manuais ou de scanners pesados e lentos de diagnóstico geral.

### 3.2. A Solução: Diagnóstico Expresso de 60 Segundos
Utilizando o FubicaDroid em um tablet ou smartphone de recepção:
1. **Conexão Instantânea:** O consultor técnico pluga o adaptador na porta OBD-II do veículo do cliente ainda na baia de entrada.
2. **Checkup de Saúde Mecânica em 1 Minuto:**
   - **Saúde da Bateria & Partida:** Medição da queda de tensão no momento em que o motor gira. Quedas abaixo de $9{,}6\text{V}$ indicam necessidade imediata de troca preventiva da bateria.
   - **Integridade do Vácuo de Admissão:** Avaliação da pressão absoluta do coletor (**MAP**) em marcha lenta estabilizada. Valores anômalos revelam entradas falsas de ar, perda de vedação em juntas ou ponto de sincronismo incorreto da distribuição.
   - **Eficiência do Termostato e Eletroventilador:** Leitura precisa da temperatura da água ($ECT$), verificando se a válvula termostática atinge e estabiliza na faixa operacional nominal ($88^\circ\text{C} - 97^\circ\text{C}$).
   - **Leitura de DTCs e Freeze Frame:** Extração instantânea de códigos de falha pendentes ou confirmados.
3. **Geração de Relatório de Recepção:** Exportação imediata de relatório em PDF/CSV para envio ao WhatsApp do cliente, agregando transparência e elevando o ticket médio de serviços preventivos.

---

## 4. Modelos de Parceria e Integração Comercial

O ecossistema FubicaDroid foi estruturado para suportar diferentes modelos de contratação B2B:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       MODELOS DE NEGÓCIO DISPONÍVEIS                        │
├──────────────────────────┬──────────────────────────┬───────────────────────┤
│    1. WHITE-LABEL APP    │      2. EMBEDDED SDK     │  3. ENTERPRISE CLOUD  │
├──────────────────────────┼──────────────────────────┼───────────────────────┤
│ • Aplicativo Android     │ • Biblioteca Kotlin/AAR  │ • API REST / Webhooks │
│   customizado com a      │   contendo o driver      │   para integração com │
│   marca da frota ou rede │   serial e o modelo      │   ERPs de frotas      │
│   de oficinas.           │   Speed-Density.         │   (Totvs, SAP, etc.). │
│ • Deploy privado via     │ • Fácil integração no    │ • Painel consolidado  │
│   Google Play Console ou │   app corporativo atual  │   Car Monitor com     │
│   MDM corporativo.       │   da empresa cliente.    │   BI multi-veículos.  │
└──────────────────────────┴──────────────────────────┴───────────────────────┘
```

---

## 5. Contato Comercial & Avaliação sob NDA

Para demonstrações ao vivo, provas de conceito (PoC) com veículos da frota ou licenciamento do motor computacional sob Acordo de Confidencialidade (NDA):

- **Responsável Técnico & Comercial:** Anderson RGS
- **E-mail:** `andersonrgs@gmail.com`
- **Repositório Vitrine:** [https://github.com/andersonrgs/FubicaDroid-Showcase](https://github.com/andersonrgs/FubicaDroid-Showcase)
- **Dashboard de Telemetria:** [https://andersonrgs.github.io/Car_dashboard/](https://andersonrgs.github.io/Car_dashboard/)
