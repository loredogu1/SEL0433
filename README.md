# SEL0433 — Projeto 3: Controle PWM e Comunicação

**Nome:** PREENCHER  
**Número USP:** PREENCHER  
**Plataforma:** ESP32 DevKit  
**Simulador:** Wokwi  
**UART:** 115200 baud  

## 1. Visão geral

O projeto foi dividido em três aplicações independentes:

1. **LED RGB com LEDC:** três canais PWM independentes, frequência de 5 kHz, resolução de 8 bits e incrementos distintos para cada cor.
2. **Servomotor por potenciômetro:** leitura do ADC e controle do ângulo com a biblioteca ESP32Servo.
3. **Aplicação própria com MCPWM:** posicionador de servomotor controlado por joystick, com OLED I2C, comunicação serial, botão de emergência com interrupção externa e LED de estado.

A organização em três pastas permite abrir, compilar e testar cada circuito separadamente no Wokwi.

---

## 2. Estrutura do repositório

```text
SEL0433_Projeto3_Entrega_Final/
├── parte1_led_rgb_ledc/
│   ├── main/main.c
│   ├── CMakeLists.txt
│   ├── main/CMakeLists.txt
│   ├── diagram.json
│   ├── sdkconfig.defaults
│   └── wokwi.toml
├── parte2_servo_potenciometro/
│   ├── sketch.ino
│   ├── libraries.txt
│   └── diagram.json
├── parte3_mcpwm_posicionador/
│   ├── main/main.c
│   ├── CMakeLists.txt
│   ├── main/CMakeLists.txt
│   ├── diagram.json
│   ├── sdkconfig.defaults
│   └── wokwi.toml
├── docs/
├── CHECKLIST_ENTREGA.md
├── ROTEIRO_VIDEO.md
└── ENTREGA_E_DISCIPLINAS.txt
```

---

# Parte 1 — Controle PWM de LED RGB

## 3. Objetivo

Controlar separadamente as componentes vermelha, verde e azul de um LED RGB de cátodo comum usando o periférico LEDC da ESP32.

## 4. Configuração

| Parâmetro | Valor |
|---|---:|
| Frequência PWM | 5 kHz |
| Resolução | 8 bits |
| Faixa do duty digital | 0 a 255 |
| Incremento vermelho | 15% |
| Incremento verde | 5% |
| Incremento azul | 10% |
| UART | 115200 baud |

## 5. Ligações

| Elemento | GPIO |
|---|---:|
| LED vermelho | GPIO25 |
| LED verde | GPIO26 |
| LED azul | GPIO27 |
| Cátodo comum | GND |

Cada terminal de cor utiliza um resistor de 220 Ω.

## 6. Funcionamento

O programa configura primeiro o temporizador LEDC e depois três canais independentes. Os percentuais são convertidos para valores entre 0 e 255:

```text
duty = percentual × 255 / 100
```

A cada 250 ms, as três cores recebem incrementos diferentes. Quando uma componente ultrapassa 100%, ela retorna ao início. A UART mostra o percentual e o duty digital de cada canal.

Exemplo:

```text
R:  45% duty=114 | G:  15% duty= 38 | B:  30% duty= 76
```

## 7. Resultado esperado

O LED apresenta uma sequência contínua de cores. Como os incrementos são diferentes, as componentes não reiniciam simultaneamente, produzindo combinações variadas.

---

# Parte 2.1 — Servomotor controlado por potenciômetro

## 8. Objetivo

Ler a posição de um potenciômetro pelo ADC da ESP32 e convertê-la em um ângulo de 0° a 180° para controlar um servomotor.

## 9. Ligações

| Elemento | GPIO |
|---|---:|
| Potenciômetro — sinal | GPIO34 |
| Servo — PWM | GPIO18 |
| Potenciômetro | 3,3 V e GND |
| Servo | 5 V e GND |

## 10. Conversão

O ADC opera com resolução de 12 bits:

```text
ADC: 0 a 4095
Ângulo: 0° a 180°
```

O pulso do servo varia aproximadamente de 500 µs a 2400 µs em um período de 20 ms, equivalente a 50 Hz.

A UART informa:

```text
ADC=2048 | Angulo=90 graus | Pulso=1450 us | Duty=7.25%
```

## 11. Biblioteca

O arquivo `libraries.txt` solicita a biblioteca:

```text
ESP32Servo
```

---

# Parte 2.2 — Aplicação própria com MCPWM

## 12. Proposta

A aplicação desenvolvida é um **posicionador de servomotor com joystick e parada de emergência**.

O eixo horizontal do joystick define o ângulo-alvo entre 0° e 180°. O eixo vertical define o passo de deslocamento, entre 1° e 10° por atualização. Dessa forma, é possível controlar tanto a posição final quanto a rapidez da resposta.

## 13. Recursos utilizados

- MCPWM para geração do sinal do servomotor;
- dois canais do ADC para leitura do joystick;
- OLED SSD1306 conectado por I2C;
- botão com interrupção externa;
- UART em 115200 baud;
- LED integrado como indicação de emergência;
- média de oito amostras do ADC;
- movimento gradual até o ângulo-alvo.

## 14. Ligações

| Elemento | GPIO |
|---|---:|
| Joystick horizontal | GPIO34 / ADC1_CH6 |
| Joystick vertical | GPIO35 / ADC1_CH7 |
| Servo PWM | GPIO18 |
| Botão de emergência | GPIO14 |
| OLED SDA | GPIO21 |
| OLED SCL | GPIO22 |
| OLED VCC | 5 V |
| LED de emergência | GPIO2 |

O botão é ligado entre GPIO14 e GND. O código habilita o resistor de pull-up interno e detecta a borda de descida.

## 15. Configuração do MCPWM

O temporizador MCPWM possui resolução de 1 MHz:

```text
1 tick = 1 µs
Período = 20.000 ticks = 20 ms
Frequência = 50 Hz
```

O comparador define a largura do pulso entre 500 µs e 2500 µs:

```text
0°   -> 500 µs
90°  -> 1500 µs
180° -> 2500 µs
```

O gerador coloca a saída em nível alto quando o contador retorna a zero e em nível baixo quando atinge o valor do comparador.

## 16. Emergência

Quando o botão é pressionado:

- a interrupção registra o evento;
- o sistema aplica debounce temporal;
- o modo de emergência é alternado;
- o servomotor retorna ao centro, em 90°;
- o LED integrado acende;
- o OLED mostra `EMERGENCIA`;
- a UART informa o novo estado.

Ao pressionar novamente, o controle pelo joystick é liberado.

## 17. OLED

O display mostra quatro linhas:

```text
MCPWM SERVO
ALVO:  90
ATUAL: 87
PASSO: 3 OK
```

Durante a parada:

```text
EMERGENCIA
```

## 18. Modelagem do movimento

O eixo horizontal define diretamente o alvo. O servo não salta imediatamente para esse valor: o ângulo atual aproxima-se gradualmente do alvo.

```text
se atual < alvo: atual = atual + passo
se atual > alvo: atual = atual - passo
```

Esse comportamento representa uma limitação de velocidade e evita movimentos abruptos.

## 19. Monitoramento serial

Exemplo:

```text
ADC_X=2048 | ADC_Y=1024 | Alvo=90 graus | Atual=87 graus |
Passo=3 | Estado=NORMAL
```

---

## 20. Como executar

### Parte 1 e Parte 3 — ESP-IDF

1. Abrir um projeto ESP-IDF no Wokwi ou no VS Code.
2. Copiar os arquivos da pasta correspondente.
3. Executar:

```bash
idf.py set-target esp32
idf.py build
```

4. Iniciar a simulação pelo Wokwi.
5. Abrir o monitor serial em 115200 baud.

### Parte 2 — Arduino

1. Criar um projeto ESP32 Arduino no Wokwi.
2. Copiar `sketch.ino`, `libraries.txt` e `diagram.json`.
3. Iniciar a simulação.
4. Girar o potenciômetro e observar o servo.

---

## 21. Testes realizados ou a registrar

### Parte 1

- duty de cada canal em 0%;
- duty de cada canal próximo de 100%;
- diferença entre os incrementos;
- comunicação serial em 115200 baud.

### Parte 2

- potenciômetro no mínimo: servo próximo de 0°;
- potenciômetro no centro: servo próximo de 90°;
- potenciômetro no máximo: servo próximo de 180°.

### Parte 3

- joystick horizontal nos dois extremos;
- alteração da velocidade pelo eixo vertical;
- atualização do OLED;
- monitoramento pela UART;
- acionamento do botão por interrupção;
- retorno do servo a 90° durante a emergência;
- LED integrado aceso na emergência.

---

## 22. Discussão dos resultados

O LEDC é adequado para o controle independente de LEDs, pois fornece vários canais PWM e permite configurar frequência e resolução de forma simples. No primeiro experimento, três canais compartilham o mesmo temporizador e mantêm duty cycles independentes.

O controle do servo por potenciômetro demonstra a integração entre ADC e PWM. A leitura analógica é convertida diretamente em posição angular, oferecendo uma interface manual intuitiva.

Na aplicação própria, o MCPWM oferece uma estrutura mais especializada, formada por temporizador, operador, comparador e gerador. Essa organização é apropriada para aplicações de controle de motores e atuadores, nas quais sincronismo, atualização de comparadores, falhas e sinais complementares podem ser necessários.

O uso de uma ESP32 de 32 bits é justificado na terceira aplicação pela execução simultânea de ADC, MCPWM, I2C, UART, interrupção externa e lógica de interface. Para um controle simples de apenas um LED ou um servo fixo, um microcontrolador mais básico poderia ser suficiente e apresentar menor custo e consumo.

## 23. Limitações

A simulação não reproduz perfeitamente ruídos elétricos, corrente do servomotor, queda de tensão ou limitações mecânicas. Em uma montagem física, o servo deve possuir alimentação adequada e terra comum com a ESP32.

## 24. Evidências a adicionar antes da entrega

Adicionar à pasta `docs/`:

1. captura da Parte 1 em execução;
2. captura do monitor serial da Parte 1;
3. captura da Parte 2 com o servo em aproximadamente 90°;
4. captura da Parte 3 em estado normal;
5. captura da Parte 3 em emergência;
6. imagem do OLED;
7. links públicos dos três projetos no Wokwi, caso sejam publicados.

## 25. Referências técnicas

- Espressif Systems — ESP-IDF Programming Guide: LED Control.
- Espressif Systems — ESP-IDF Programming Guide: MCPWM.
- Espressif Systems — ESP-IDF Programming Guide: ADC Oneshot.
- Espressif Systems — ESP-IDF Programming Guide: I2C.
- Espressif Systems — exemplo oficial de controle de servomotor por MCPWM.
- Documentação da biblioteca ESP32Servo.
