# Aferidor de Temperatura de Forno Industrial (PIC18)

## Sobre o projeto

O projeto visa projetar e implementar um dispositivo de aferição de temperatura e tempo para um forno industrial, utilizando um microcontrolador da família PIC18 (especificamente o modelo PIC18F4550). Esse sistema tem a finalidade de monitorar a temperatura interna de um forno em intervalos de tempo, uma funcionalidade aplicada em processos de fabricação metálica, pintura eletrostática e manipulação de componentes químicos.
O funcionamento do sistema consiste em realizar a leitura da temperatura na faixa de 0°C a 100°C por meio de um sensor LM35 e exibir esse valor continuamente no display LCD. O dispositivo conta com botões para selecionar tempos predeterminados de medição, entre aferição de curta ou longa duração, exibindo a contagem regressiva correspondente. Além disso, o projeto faz uso de sinalização luminosa via LEDs para simular o estado da resistência do forno em temperaturas elevadas e medidas de prevenção contra o efeito bouncing dos botões mecânicos. A implementação foi realizada via para a simulação no ambiente SimulIDE e execução na placa de desenvolvimento Kit EasyPIC v7.
## Objetivos

Os principais objetivos para esse projeto são:

* Compreender a interação entre os elementos e os circuitos internos dos microcontroladores da família PIC18.
* Simular o comportamento de hardware no ambiente SimulIDE integrando periféricos externos como displays LCD 16x2 em modo de 4 bits, botões e LEDs.
* Desenvolver firmware em linguagem C modularizado aplicando técnicas como tratamento de debounce e cálculo de bases de tempo de timers.


### <ins>Checkpoint 1 – Tratamento do efeito bouncing e acionamento do display LCD<ins>

Para o primeiro *Checkpoint*, o foco principal foi o acionamento do display LCD e o tratamento via software do efeito bouncing (repique). O bouncing é um problema comum em chaves mecânicas, fazendo com que o microcontrolador interprete erroneamente múltiplos acionamentos elétricos ao invés de apenas um.

O desenvolvimento consistiu na criação de um firmware que exibe a frase **"HelloWrld"** na primeira linha de um display LCD operando como uma interface de comunicação de 4 bits. Na segunda linha, foi estruturado um contador de ciclo único de 0 a 9 que é incrementado toda vez que o botão é pressionado. Utilizamos uma flag auxiliar de software e detecção por borda de subida para garantir que o repique dos contatos metálicos do botão fosse totalmente filtrado pelo código, proporcionando uma leitura limpa.

```c
sbit LCD_RS at RB4_bit;
sbit LCD_EN at RB5_bit;
sbit LCD_D4 at RB0_bit;
sbit LCD_D5 at RB1_bit;
sbit LCD_D6 at RB2_bit;
sbit LCD_D7 at RB3_bit;

sbit LCD_RS_Direction at TRISB4_bit;
sbit LCD_EN_Direction at TRISB5_bit;
sbit LCD_D4_Direction at TRISB0_bit;
sbit LCD_D5_Direction at TRISB1_bit;
sbit LCD_D6_Direction at TRISB2_bit;
sbit LCD_D7_Direction at TRISB3_bit;

sbit BOTAO at RD0_bit;
sbit BOTAO_Direction at TRISD0_bit;

unsigned short contador = 0;
unsigned short botao_flag = 0;
char digito[2];

void atualiza_lcd() {
    digito[0] = contador + '0';
    digito[1] = '\0';

    Lcd_Out(1, 1, "HelloWrld");
    Lcd_Out(2, 1, "Contador: ");
    Lcd_Out(2, 11, digito);
}

void main() {
    ADCON1 = 0x0F;
    CMCON = 0x07;

    TRISB = 0x00;
    BOTAO_Direction = 1;

    Lcd_Init();
    Lcd_Cmd(_LCD_CLEAR);
    Lcd_Cmd(_LCD_CURSOR_OFF);

    atualiza_lcd();

    while(1) {
        if (BOTAO == 1 && botao_flag == 0) {
            Delay_ms(30);

            if (BOTAO == 1) {
                botao_flag = 1;

                contador++;

                if (contador > 9) {
                    contador = 0;
                }

                atualiza_lcd();
            }
        }

        if (BOTAO == 0) {
            botao_flag = 0;
        }
    }
}
```

<img width="423" height="364" alt="Circuito - Checkpoint 1" src="https://github.com/user-attachments/assets/f6abc80e-d822-4b97-918a-bcb5dd7eb76d" />




### <ins>Checkpoint 2 – Contagem de tempo utilizado Timers e interrupções no PIC <ins>

Para o segundo *Checkpoint*, desenvolvemos a base de temporização para o contador regressivo. Nesta etapa, implementamos a manipulação de dois timers em conjunto com interrupções externas acionadas por dois botões push buttons distintos.

A lógica aplicada integra o microcontrolador operando com um cristal oscilador de 8 MHz. Utilizamos o temporizador TMR0 configurado com interrupção externa para ser a base de tempo do intervalo de contagem de longa duração (que dura 60 segundos com precisão na casa de 1 segundo). Simultaneamente, o temporizador TMR1 foi configurado para gerenciar o intervalo de curta duração (10 segundos, com base de 250 milissegundos). Todos os dados processados na contagem regressiva passaram a ser enviados para exibição em uma das linhas do display LCD inicializado no checkpoint anterior, consolidando a rotina principal focada em eventos.

```c
sbit LCD_RS at RB4_bit;
sbit LCD_EN at RB5_bit;
sbit LCD_D4 at RB0_bit;
sbit LCD_D5 at RB1_bit;
sbit LCD_D6 at RB2_bit;
sbit LCD_D7 at RB3_bit;

sbit LCD_RS_Direction at TRISB4_bit;
sbit LCD_EN_Direction at TRISB5_bit;
sbit LCD_D4_Direction at TRISB0_bit;
sbit LCD_D5_Direction at TRISB1_bit;
sbit LCD_D6_Direction at TRISB2_bit;
sbit LCD_D7_Direction at TRISB3_bit;

sbit BOTAO_60 at RD0_bit;
sbit BOTAO_10 at RD1_bit;

unsigned short tempo = 0;
unsigned short modo = 0; 
unsigned short rodando = 0;
unsigned short atualiza = 1;

unsigned short ticks_tmr1 = 0; 

char estado_b0 = 0;
char estado_b1 = 0;

void mostra_lcd() {
    char txt[4];

    txt[0] = (tempo / 10) + '0';
    txt[1] = (tempo % 10) + '0';
    txt[2] = '\0';
    
    if (modo == 1) 
    {
        Lcd_Out(1, 1, "Modo: 60s       ");
        Lcd_Out(2, 1, "Tempo: ");
        Lcd_Out(2, 8, txt);
        Lcd_Out(2, 10, "s       "); 
    } 
    
    else if (modo == 2)
    {
        Lcd_Out(1, 1, "Modo: 10s       ");
        Lcd_Out(2, 1, "Tempo: ");
        Lcd_Out(2, 8, txt);
        Lcd_Out(2, 10, "s       "); 
    } 
    
    else
    {
        Lcd_Out(1, 1, "Aperte um botao");
        Lcd_Out(2, 1, "para iniciar...");
    }


}

void inicia_60s() {
    tempo = 60;
    modo = 1;
    rodando = 1;
    T1CON.TMR1ON = 0; 
    TMR0H = 0xC2;
    TMR0L = 0xF7;
    INTCON.TMR0IF = 0;
    T0CON.TMR0ON = 1;
    atualiza = 1;
}

void inicia_10s() {
    tempo = 10;
    modo = 2;
    rodando = 1;
    ticks_tmr1 = 0;

    T0CON.TMR0ON = 0; 
    TMR1H = 0x0B;
    TMR1L = 0xDC;

    PIR1.TMR1IF = 0;
    T1CON.TMR1ON = 1;

    atualiza = 1;
}

void interrupt() {
    if (INTCON.TMR0IF) {
        TMR0H = 0xC2;
        TMR0L = 0xF7;
        INTCON.TMR0IF = 0;

        if (rodando && tempo > 0) {
            tempo--;
            atualiza = 1;

            if (tempo == 0) {
                rodando = 0;
                T0CON.TMR0ON = 0;
            }
        }
    }

    if (PIR1.TMR1IF) {
        TMR1H = 0x0B;
        TMR1L = 0xDC;
        PIR1.TMR1IF = 0;

        if (rodando && tempo > 0) {
            ticks_tmr1++;

            if (ticks_tmr1 >= 4) {
                ticks_tmr1 = 0;
                tempo--;
                atualiza = 1;

                if (tempo == 0) {
                    rodando = 0;
                    T1CON.TMR1ON = 0;
                }
            }
        }
    }
}

void main() {
    ADCON1 = 0x0F; 
    CMCON = 0x07;  

    TRISB = 0x00; 

    TRISD0_bit = 1; 
    TRISD1_bit = 1; 

    Lcd_Init();
    Lcd_Cmd(_LCD_CLEAR);
    Lcd_Cmd(_LCD_CURSOR_OFF);
    
  
    
    T0CON = 0b00000110;
    INTCON.TMR0IE = 1;
    INTCON.TMR0IF = 0;
    
  
    
    T1CON = 0b10110000;
    PIE1.TMR1IE = 1;
    PIR1.TMR1IF = 0;

    INTCON.PEIE = 1; 
    INTCON.GIE = 1; 

    mostra_lcd();

    while (1) {
        if (BOTAO_60 == 1 && estado_b0 == 0) {
            Delay_ms(30);
            if (BOTAO_60 == 1) {
                estado_b0 = 1;
                inicia_60s();
            }
        } else if (BOTAO_60 == 0) {
            estado_b0 = 0;
        }

        if (BOTAO_10 == 1 && estado_b1 == 0) {
            Delay_ms(30);
            if (BOTAO_10 == 1) {
                estado_b1 = 1;
                inicia_10s();
            }
        } else if (BOTAO_10 == 0) {
            estado_b1 = 0;
        }

        if (atualiza) {
            atualiza = 0;
            mostra_lcd();
        }
    }
}

```
<img width="618" height="440" alt="Circuito - Checkpoint 2" src="https://github.com/user-attachments/assets/acb40645-ff3a-45b4-9227-614e86a4e838" />

## <ins>Entrega Final – Seleção da Temperatura <ins>

Por fim, na entrega final, integramos todos os módulos anteriores ao subsistema de conversão Analógico-Digital (ADC) para criar a lógica de aferição da temperatura. A medição no projeto simula as saídas do sensor LM35 na faixa de 0°C a 100°C usando um potenciômetro ligado aos pinos analógicos do PIC18F4550.

O destaque para esta fase está na configuração dedicada do registrador ADCON1. Como o ADC do PIC tem resolução de 10 bits e o sensor LM35 fornece pequenas variações de tensão, utilizamos uma alimentação de referência externa de 1V ao invés dos convencionais 5V na porta AN3 (RA3) e essa mesma alimentação de 1V para o potenciômetro, o que permitiu mantar a faixa de tensão máxima do potênciometro entre 0V e 1 V, o que aumentou a precisão da simulação do LM35. Após o ADC fazer a leitura, o valor processado é formatado para exibir 3 algarismos com 1 casa decimal no display ("XX.X °C") porém, ao invés do valor armazenado no código ser de fato um número de ponto flutuante (float) ele é um número inteiro, o que dificulta um pouco a implementação do circuito, porém reduz o uso de memória.

Além disso, mudamos um pouco a funcionalidade dos 2 botões implementados no checkpoint2. agora, ao invés de cada um dos botões servir para escolher um modo de contagem regressiva diferente, um deles, atua como acionador geral de todo o processo simultâneo, ou seja, quando é acionado, ele liga a contagem selecionada. Já o segundo botão alterna entre os dois modos de contagem existentes (curta, de 10 segundos e longa de 60 segundo). Se a temperatura do sistema ultrapassar os 50°C, um LED foi configurado para acender em resposta, indicando que a resistência interna do forno está ativa. Normalmente colocariamos a resistência conectada a esse LED em 330 ohms para reduzir sua luminância a um nível aceitável e aumentar o tempo de vida do diodo. No entanto, ao fazer isso no Simulide o LED fica muito apagado. Dessa forma, para poder gerar a imagem do circuito de maneira mais lúdica utilizamos o resistor com valor de 100 ohms, que deixa a cor amarela do LED bem mais destacada.

Segue abaixo o código completo implementado no MikroC para a realização desse projeto. Os arquivos de simulação do Simulide, o código Hex colocado no PIC 18f4550 do simulador e o Código C, junto com um documento completo desse projeto podem ser encontrados em: https://github.com/Franc0liv/SEL0433/tree/main/Projeto_2/Misc

```c
// Conexões do Módulo LCD
sbit LCD_RS at RB4_bit;
sbit LCD_EN at RB5_bit;
sbit LCD_D4 at RB0_bit;
sbit LCD_D5 at RB1_bit;
sbit LCD_D6 at RB2_bit;
sbit LCD_D7 at RB3_bit;

sbit LCD_RS_Direction at TRISB4_bit;
sbit LCD_EN_Direction at TRISB5_bit;
sbit LCD_D4_Direction at TRISB0_bit;
sbit LCD_D5_Direction at TRISB1_bit;
sbit LCD_D6_Direction at TRISB2_bit;
sbit LCD_D7_Direction at TRISB3_bit;

// Definição de Botões e LED
sbit BOTAO_SEL at RD0_bit;    // Seleciona o tempo (Curto ou Longo)
sbit BOTAO_START at RD1_bit;  // Inicia a contagem
sbit LED_AQUEC at RD2_bit;    // LED de Resistência (> 50 °C)

// Variáveis Globais
unsigned short tempo_sel = 10; // Modo selecionado (10 ou 60)
unsigned short tempo = 0;      // Tempo atual da contagem regressiva
unsigned short rodando = 0;    // Flag de status (1 = contando)
unsigned short atualiza = 1;   // Flag para atualizar o display
unsigned short ticks_tmr1 = 0;

char estado_b_sel = 0;
char estado_b_start = 0;

unsigned int adc_value = 0;
unsigned int temp_int = 0;
unsigned int temp_int_old = 0xFFFF; // Força primeira atualização do LCD

void inicia_60s() {
    tempo = 60;
    rodando = 1;
    T1CON.TMR1ON = 0;

    // Configura Timer0 para 1 segundo exato (Prescaler 1:128)
    TMR0H = 0xC2;
    TMR0L = 0xF7;
    INTCON.TMR0IF = 0;
    T0CON.TMR0ON = 1;
    atualiza = 1;
}

void inicia_10s() {
    tempo = 10;
    rodando = 1;
    ticks_tmr1 = 0;
    T0CON.TMR0ON = 0;

    // Configura Timer1 para 0.25 segundos (Prescaler 1:8, precisará de 4 ticks)
    TMR1H = 0x0B;
    TMR1L = 0xDC;
    PIR1.TMR1IF = 0;
    T1CON.TMR1ON = 1;
    atualiza = 1;
}

void interrupt() {
    // Interrupção Timer0 (Usado para 60s)
    if (INTCON.TMR0IF) {
        TMR0H = 0xC2;
        TMR0L = 0xF7;
        INTCON.TMR0IF = 0;

        if (rodando && tempo > 0) {
            tempo--;
            atualiza = 1;
            if (tempo == 0) {
                rodando = 0;
                T0CON.TMR0ON = 0; // Desliga Timer0 ao fim
            }
        }
    }

    // Interrupção Timer1 (Usado para 10s)
    if (PIR1.TMR1IF) {
        TMR1H = 0x0B;
        TMR1L = 0xDC;
        PIR1.TMR1IF = 0;

        if (rodando && tempo > 0) {
            ticks_tmr1++;
            if (ticks_tmr1 >= 4) { // 4 x 0.25s = 1s
                ticks_tmr1 = 0;
                tempo--;
                atualiza = 1;

                if (tempo == 0) {
                    rodando = 0;
                    T1CON.TMR1ON = 0; // Desliga Timer1 ao fim
                }
            }
        }
    }
}

void mostra_lcd() {
    char txt_tempo[4];
    char txt_temp[8];

    // Formatação do tempo
    txt_tempo[0] = (tempo / 10) + '0';
    txt_tempo[1] = (tempo % 10) + '0';
    txt_tempo[2] = '\0';

    // Formatação da temperatura em formato "XX.X" usando apenas inteiros
    // temp_int varia de 0 a 1000
    if (temp_int >= 1000) {
        txt_temp[0] = '1'; txt_temp[1] = '0'; txt_temp[2] = '0';
        txt_temp[3] = '.'; txt_temp[4] = '0'; txt_temp[5] = '\0';
    } else {
        txt_temp[0] = (temp_int / 100) ? ((temp_int / 100) + '0') : ' '; // Esconde zero à esquerda
        txt_temp[1] = ((temp_int / 10) % 10) + '0';
        txt_temp[2] = '.';
        txt_temp[3] = (temp_int % 10) + '0';
        txt_temp[4] = '\0';
    }

    // Linha 1: Exibe a Temperatura
    Lcd_Out(1, 1, "Temp: ");
    Lcd_Out(1, 7, txt_temp);
    Lcd_Chr(1, 12, 223); // Imprime o símbolo de grau (°)
    Lcd_Out(1, 13, "C   ");

    // Linha 2: Exibe estado do Temporizador
    if (rodando) {
        Lcd_Out(2, 1, "Restam: ");
        Lcd_Out(2, 9, txt_tempo);
        Lcd_Out(2, 11, "s     ");
    } else {
        if (tempo_sel == 10) {
            Lcd_Out(2, 1, "Modo: 10s (Curt)");
        } else {
            Lcd_Out(2, 1, "Modo: 60s (Long)");
        }
    }
}


void main() {
     // Inicializamos o módulo ADC do MikroC primeiro para não dar problema
    // Isso é: Evita que a biblioteca sobrescreva o ADCON1 depois
    ADC_Init();

    // Daí sim configuramos o ADCON1 apenas após a biblioteca ter sido iniciada
    // VCFG1 (bit 5): 0 (VSS como Vref-)
    // VCFG0 (bit 4): 1 (AN3/RA3 como Vref+ de 1V)
    // PCFG (bits 3 a 0): 1011 (AN0, AN1, AN2 e AN3 como analógicos)
    ADCON1 = 0x1B;

    CMCON = 0x07; // Desliga comparadores

    // Configuração dos pinos (TRIS)
    TRISB = 0x00;     // Porta B como saída (LCD)
    TRISD0_bit = 1;   // RD0: Botão de Seleção (Entrada)
    TRISD1_bit = 1;   // RD1: Botão Start (Entrada)
    TRISD2_bit = 0;   // RD2: LED de Aquecimento (Saída)
    LED_AQUEC = 0;

    TRISA0_bit = 1;   // RA0/AN0: Entrada analógica LM35
    TRISA3_bit = 1;   // RA3/AN3: Entrada tensão Vref+ (1V)

    // Inicializa o LCD
    Lcd_Init();
    Lcd_Cmd(_LCD_CLEAR);
    Lcd_Cmd(_LCD_CURSOR_OFF);
    // Configurações dos Timers e Interrupções
    T0CON = 0b00000110;
    INTCON.TMR0IE = 1;
    INTCON.TMR0IF = 0;

    T1CON = 0b10110000;
    PIE1.TMR1IE = 1;
    PIR1.TMR1IF = 0;

    INTCON.PEIE = 1;
    INTCON.GIE = 1;

    tempo = tempo_sel;
    mostra_lcd();

    while (1) {
        // Leitura do ADC
        // ATENÇÃO: É vital usar ADC_Get_Sample(0) em vez de ADC_Read(0).
        // ADC_Read() força internamente a Vref para os padrões do microcontrolador (5V),
        // anulando nossa configuração do ADCON1, enquanto ADC_Get_Sample() respeita o nosso setup!
        adc_value = ADC_Get_Sample(0);

        // Converte valor ADC (0 a 1023) para "0 a 1000" para ter 1 casa decimal sem usar float.
        // 1023 -> 100.0 graus -> representado em int como 1000.
        temp_int = (unsigned int)(((unsigned long)adc_value * 1000) / 1023);

        // Controle da resistência/forno: Acende LED se > 50.0 °C
        if (temp_int > 500) {
            LED_AQUEC = 1;
        } else {
            LED_AQUEC = 0;
        }

        // Apenas pede para atualizar o LCD se a temperatura mudou
        if (temp_int != temp_int_old) {
            temp_int_old = temp_int;
            atualiza = 1;
        }

        // Lógica: Botão de Seleção de Modo (Atua apenas se parado)
        if (BOTAO_SEL == 1 && estado_b_sel == 0) {
            Delay_ms(30); // Debounce
            if (BOTAO_SEL == 1) {
                estado_b_sel = 1;
                if (!rodando) {
                    if (tempo_sel == 10) tempo_sel = 60;
                    else tempo_sel = 10;
                    tempo = tempo_sel;
                    atualiza = 1;
                }
            }
        } else if (BOTAO_SEL == 0) {
            estado_b_sel = 0;
        }

        // Lógica: Botão Start
        if (BOTAO_START == 1 && estado_b_start == 0) {
            Delay_ms(30); // Debounce
            if (BOTAO_START == 1) {
                estado_b_start = 1;
                if (!rodando) {
                    if (tempo_sel == 60) inicia_60s();
                    else inicia_10s();
                }
            }
        } else if (BOTAO_START == 0) {
            estado_b_start = 0;
        }

        // Rotina de impressão no display (Evita piscar o LCD em loop infinito)
        if (atualiza) {
            atualiza = 0;
            mostra_lcd();
        }

        Delay_ms(20); // Pequeno atraso para manter estabilidade do loop
    }
}
```
<img width="968" height="616" alt="WhatsApp Image 2026-06-23 at 00 08 55" src="https://github.com/user-attachments/assets/77a5fdcd-ef73-4ea4-9362-2031314e2973" />

## Autores
| Nome | NUSP |
| --- | --- |
|Raphael Franco de Oliveira	| 13862393|
|Giulliano Olivato da Silva	| 9944204|
|Guilherme Pereira Loredo   | 11885190|


