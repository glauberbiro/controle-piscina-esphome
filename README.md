# Controle Piscina (ESPHome)

> ⚠️ **Projeto em teste.** Ainda está na bancada (ESP32 de teste, sensores NTC ainda não instalados) e não substituiu o Tholz MMZ na piscina. As seguranças ainda não foram testadas com temperaturas reais. Use por sua conta e risco e mantenha as proteções elétricas (relé térmico e DR).

Controlador diferencial de aquecimento solar de piscina em **ESPHome**, feito para substituir um **Tholz MMZ1195N** — e que também comanda a **bomba do filtro**. Toda a lógica do aquecimento roda no próprio ESP32: continua funcionando sem Wi-Fi ou Home Assistant.

- 2 sensores NTC 10K (piscina e coletores) — compatível com os sensores originais da Tholz (B = 3980)
- 2 relés acionando contatores (bomba de aquecimento e bomba do filtro)
- Display OLED 128×64, LED de status, botão físico
- Todos os parâmetros ajustáveis pelo Home Assistant e salvos no ESP

## Arquivos

| Arquivo | O que é |
|---|---|
| [`esphome/controle-piscina.yaml`](esphome/controle-piscina.yaml) | Placa atual: ESP32 DevKit + OLED SSD1306 128×64 |
| [`esphome/controle-piscina-s3.yaml`](esphome/controle-piscina-s3.yaml) | Placa definitiva: Waveshare ESP32-S3-Touch-LCD-2 (tela LVGL com touch, mesmos recursos da ESP32) — ainda não testada na placa |
| [`esphome/secrets.yaml.example`](esphome/secrets.yaml.example) | Modelo dos segredos (Wi-Fi, chave da API, senha do OTA) |
| [`homeassistant/automacao-avisos-reparos.yaml`](homeassistant/automacao-avisos-reparos.yaml) | Automação que publica falhas em Configurações → Reparos (requer Spook) |

## Lógica do aquecimento (ordem de prioridade)

1. **Anticongelamento** — coletor ≤ 3 °C → liga, em qualquer modo
2. **Modo Desligado** → desliga
3. **Sobreaquecimento do coletor** — coletor ≥ 70 °C → desliga na hora (também no modo Ligado); volta 2 °C abaixo. Protege canos de PVC
4. **Modo Ligado** → liga, mas volta sozinho para Automático:
   - depois do tempo máximo (6 h)
   - se a piscina chegar na temperatura máxima (proteção ligada por padrão; protege piscina de vinil)
5. **Falha de sensor** (aberto/curto) → desliga
6. **Piscina ≥ temperatura máxima** (31 °C, histerese 0,5 °C) → desliga
7. **Diferencial** — liga com coletor − piscina ≥ 7 °C, desliga com ≤ 3 °C; tempo mínimo de 3 min entre trocas

Os valores entre parênteses são os padrões; todos são ajustáveis no Home Assistant.

## Ligações — ESP32 DevKit

| Componente | Pino do componente | ESP32 |
|---|---|---|
| NTC piscina | 10k 1% do 3V3 ao ponto do meio · ponto do meio · outra ponta do NTC | 3V3 · IO34 · GND |
| NTC coletor | idem | 3V3 · IO35 · GND |
| (opcional) 100 nF | de cada IO do NTC ao GND (recomendado com cabo longo) | — |
| Relé 1 — aquecimento (jumper em H) | IN · VCC · GND | IO26 · 5V · GND |
| Relé 2 — filtro (jumper em H) | IN · VCC · GND | IO27 · 5V · GND |
| OLED 128×64 I2C | VCC · GND · SCL · SDA | 3V3 · GND · IO22 · IO21 |
| Botão | um terminal · outro | IO14 · GND |
| LED da placa | — | GPIO2 |

- **Contatores:** COM + NO de cada relé na bobina; snubber RC (100 Ω + 100 nF X2 275 VAC) em A1/A2.
- **Elétrica:** use relé térmico na bomba e disjuntor DR (NBR 5410). O ESP não substitui essas proteções.
- **Diagnóstico** (multímetro no IO do NTC): ~1,6 V ok · ~3,3 V NTC aberto · ~0 V curto ou resistor sem 3V3.

## Botão e LED

- Toque curto: acende a tela por 60 s · segurar 2 s: troca o modo (Automático → Ligado → Desligado)
- LED pisca lento: Wi-Fi e Home Assistant conectados · pisca rápido: sem rede

## Placa Waveshare ESP32-S3-Touch-LCD-2

| Função | GPIO | Pino (numeração oficial 1–28) |
|---|---|---|
| NTC piscina | GPIO6 | 26 |
| NTC coletor | GPIO7 | 20 |
| Relé aquecimento | GPIO17 | 24 |
| Relé filtro | GPIO18 | 23 |
| 5V dos relés | 5V | 15 |
| GND dos relés | GND | 16 |
| 3V3 dos divisores | 3V3 | 1 |
| GND dos NTC | GND | 2 |

Não use GPIO47/48 (pinos 5 e 6): são do touch. Pinagem oficial da placa: [wiki da Waveshare/Spotpear](https://spotpear.com/wiki/ESP32-S3-Touch-LCD-2-User-Guide.html).

Na tela: temperaturas, diferença, Wi-Fi, estado, caixas de **aquecimento** (indicação) e **filtro** (tocar liga/desliga) e os botões de modo AUTO / LIGAR / DESL. A tela apaga após 2 min sem toque; um toque (ou o botão "Acender tela" do HA) acorda.

Manual do Tholz MMZ e esquema da Waveshare: veja os sites dos fabricantes ([tholz.com.br](http://www.tholz.com.br), [Waveshare/Spotpear wiki](https://spotpear.com/wiki/ESP32-S3-Touch-LCD-2-User-Guide.html)).
