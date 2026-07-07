# Projeto 3: Controle PWM e Comunicação - SEL0433

**Autor:** Guilherme Pereira Loredo  
**Instituição:** Escola de Engenharia de São Carlos (EESC-USP)  
**Número USP:** [Inserir seu Número USP]  

## 📌 Objetivos do Projeto
Este projeto explora conceitos de comunicação serial e modulação por largura de pulso (PWM) aplicados a um microcontrolador de 32 bits. Desenvolvido em linguagem C para a placa ESP32 DevKit, o sistema teve sua montagem, validação e execução integralmente realizadas no ambiente de simulação Wokwi.

O controle embarcado divide-se em duas frentes principais:

Controle Luminoso: Ajuste contínuo e independente da intensidade de um LED RGB (catodo comum) via canais PWM, com monitoramento de dados em tempo real pela interface serial (UART).

Acionamento de Atuadores: Controle posicional de um servomotor a partir da leitura analógica de um potenciômetro (ADC). A aplicação evolui para o controle avançado de motores utilizando bibliotecas nativas, exibição de parâmetros via display OLED (barramento I2C) e integração com outros recursos de hardware e software.

A documentação técnica detalhada — incluindo decisões de projeto, inicialização de periféricos, estruturação de tarefas e cálculos de duty cycle — encontra-se nos comentários dos arquivos de código-fonte deste repositório.

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
