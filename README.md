# Potenciômetro Digital (Digipot) Emulado com Acoplamento Óptico e DMA no STM32F407VET6

Este repositório contém o desenvolvimento de um Potenciômetro Digital (Digipot) customizado, implementado através de um acoplamento óptico isolado (Vactrol caseiro) utilizando um LED e um LDR (Light Dependent Resistor). O projeto utiliza o microcontrolador STM32F407VET6 para controlar o brilho do LED via DAC, ler a variação de resistência via ADC, atualizar dinamicamente a curva de calibração (Look-Up Table) em tempo real via UART com DMA, e exibir os dados em um display OLED SSD1306.

Como alternativa à falta de um conversor USB-TTL dedicado, foi implementado um firmware de bypass em um ESP32 para atuar como ponte de comunicação serial transparente entre o computador e a placa STM32.

---

## Funcionalidades

- Controle analógico preciso com resolução de 12 bits no DAC para o controle do brilho do LED e 12 bits no ADC para leitura do divisor de tensão do LDR.
- Linearização por Look-Up Table (LUT), compensando a resposta exponencial natural do LDR através de um mapeamento reconfigurável de 256 posições para a palavra digital `res` (`uint8_t`).
- Calibração dinâmica via DMA com transferência de uma nova tabela de calibração de 512 bytes em segundo plano via UART, acionada por interrupção de hardware (botão K0).
- Interface visual otimizada utilizando display SSD1306 via I2C, exibindo resistência mínima e máxima, tensão instantânea do ADC e valor atual do Digipot.
- Ferramentas auxiliares em Python para geração da curva matemática e envio automatizado do arquivo binário via linha de comando.

---

## Arquitetura de Hardware e Conexões

### Componentes Utilizados

- Microcontrolador STM32F407VET6 ("Black Board")
- Módulo ESP32 utilizado exclusivamente como ponte USB-TTL
- Display OLED SSD1306 128x64 via I2C
- 1x LDR de 5 mm
- 1x LED de alto brilho de 5 mm
- 1x Resistor de 10 kΩ para o divisor de tensão
- 1x Resistor de 220 Ω para limitação de corrente do LED

### Pinagem do Sistema

| Periférico | Pino STM32F407 | Conexão / Destino | Descrição |
|---|---|---|---|
| GND | GND | Linha de Terra Comum | Referência elétrica do sistema |
| VCC | 3.3V | Linha positiva da protoboard | Alimentação do circuito |
| DAC_OUT1 | PA4 | Resistor 220 Ω -> Anodo LED | Controle de intensidade luminosa |
| ADC1_IN1 | PA1 | Divisor de tensão do LDR | Leitura analógica da resistência |
| I2C1_SCL | PB6 | SCL do SSD1306 | Clock da interface I2C |
| I2C1_SDA | PB7 | SDA do SSD1306 | Dados da interface I2C |
| USART1_TX | PA9 | RX2 GPIO16 do ESP32 | Telemetria serial |
| USART1_RX | PA10 | TX2 GPIO17 do ESP32 | Recepção da LUT |
| GPIO_Input | PE4 | Botão K0 | Entrada para modo de calibração |

---

## Estrutura do Firmware

O firmware foi desenvolvido utilizando a biblioteca HAL no STM32CubeIDE. Toda a documentação interna e comentários técnicos foram padronizados em inglês para compatibilidade internacional.

### Inicialização e Configuração do DMA

Quando o botão K0 é pressionado, a UART1 entra em modo de recepção via DMA para receber exatamente 512 bytes contendo os 256 valores de 16 bits da nova tabela de calibração.

```c
// DMA RX Complete Callback - Fires automatically when 512 bytes arrive
void HAL_UART_RxCpltCallback(UART_HandleTypeDef *huart)
{
    if (huart->Instance == USART1)
    {
        dma_rx_complete = 1; // Signal the main loop that new LUT arrived
    }
}
```

### Cálculo de Resistência Otimizado

A equação do divisor de tensão foi implementada com proteção contra divisões por zero e condições extremas de operação.

```c
float get_resistance_for_dac(uint16_t dac_val)
{
    HAL_DAC_SetValue(&hdac, DAC_CHANNEL_1, DAC_ALIGN_12B_R, dac_val);
    HAL_Delay(100); // Allow physical LDR inertia to settle

    HAL_ADC_Start(&hadc1);
    HAL_ADC_PollForConversion(&hadc1, 10);

    uint32_t adc_raw = HAL_ADC_GetValue(&hadc1);

    current_voltage = ((float)adc_raw * VCC) / 4095.0f;

    if (current_voltage >= VCC)
        current_voltage = 3.29f;

    if (current_voltage <= 0.0f)
        current_voltage = 0.01f;

    return R_FIXO * ((VCC / current_voltage) - 1.0f);
}
```

---

## Ferramentas de Calibração em Python

A pasta `/tools` contém scripts auxiliares responsáveis pela modelagem e transmissão da curva de calibração.

### `gerador_lut.py`

Responsável por:

- Gerar curvas exponenciais do tipo `f(x) = x^n`
- Compensar a não linearidade natural do LDR
- Empacotar os dados em formato binário Little-Endian
- Produzir o arquivo `lut_digipot.bin` contendo 512 bytes

### `enviador_serial.py`

Responsável por:

- Abrir automaticamente a porta serial do ESP32
- Transmitir o arquivo binário via UART
- Garantir sincronização da transferência da LUT

---

## Como Executar o Projeto

### 1. Gravar o ESP32

Compile e grave o firmware localizado em `/esp32_bypass` utilizando a Arduino IDE para transformar o ESP32 em uma ponte serial transparente.

### 2. Montar o Hardware

Monte o circuito seguindo a tabela de pinagem. O LED e o LDR devem permanecer completamente isolados da luz externa dentro da câmara óptica.

### 3. Compilar e Gravar a STM32

Abra o projeto no STM32CubeIDE, compile o firmware e grave utilizando um ST-Link.

Após a inicialização, o display OLED exibirá a curva linear padrão.

### 4. Atualizar a Curva de Calibração

Pressione o botão K0 na STM32. O display exibirá:

```text
Aguardando PC...
```

No computador, execute:

```bash
pip install pyserial
python tools/enviador_serial.py
```

Após a transferência da LUT, o display exibirá:

```text
Nova LUT OK!
```

O Digipot passará imediatamente a operar utilizando a nova curva calibrada.

---

## Análise de Resultados e Relatório de Testes

Os testes foram realizados utilizando 10 valores discretos no intervalo `res [0, 255]`, alternados ciclicamente a cada 2.5 segundos.

As medições foram feitas manualmente com multímetro nas escalas de 200 kΩ e 2000 kΩ para validação da resposta resistiva do sistema.

Os dados coletados foram utilizados para cálculo do erro RMS (Root Mean Square Error), conforme especificado pelos requisitos acadêmicos do laboratório.

---

[![Assistir Vídeo do Projeto](https://img.shields.io/badge/YouTube-Assistir%20Vídeo-red?style=for-the-badge&logo=youtube)](https://www.youtube.com/watch?v=OIEYp3sDpnQ)
