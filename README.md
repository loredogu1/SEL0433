# Projeto 3: Controle PWM e Comunicação - SEL0433

**Autor:** Guilherme Pereira Loredo  
**Instituição:** Escola de Engenharia de São Carlos (EESC-USP)  
**Número USP:** [Inserir seu Número USP]  

## 📌 Objetivos do Projeto
Este repositório contém as implementações desenvolvidas para o Projeto 3 da disciplina de Aplicação de Microprocessadores. O foco principal é a utilização do microcontrolador de 32 bits ESP32 para o controle de periféricos através de Modulação por Largura de Pulso (PWM), explorando tanto a biblioteca de controle de LEDs (`LEDC`) quanto o controle direto de hardware para acionamento de motores, aliados à conversão analógico-digital (ADC).

---

## 💻 Parte 1: Controle PWM de LED RGB

Na primeira etapa, implementou-se o controle de brilho de um LED RGB (catodo comum) variando o *duty cycle* continuamente de 0% a 100%.

### Bibliotecas e Implementação
Foi utilizada a biblioteca nativa **LEDC** da arquitetura do ESP32 para gerar um sinal PWM com resolução de 8 bits e frequência de 5 kHz. O código foi desenvolvido com diretivas de pré-processador (`#if defined(ESP_ARDUINO_VERSION_MAJOR)`) para garantir retrocompatibilidade entre as versões 2.x e 3.x do *core* do Arduino.

**Trecho de Código em Destaque (Incremento Assíncrono):**
Para garantir que cada cor variasse com uma taxa independente (Vermelho: 15, Verde: 5, Azul: 10), foi criada uma lógica de incremento em *loop* circular:

```cpp
int advancePercent(int currentValue, int stepValue) {
  if (currentValue >= 100) return 0;
  int nextValue = currentValue + stepValue;
  if (nextValue > 100) nextValue = 100;
  return nextValue;
}
