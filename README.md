#include <stdio.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_log.h"

#include "HCF_IOTEC.h"   // Vai se tornar HCF_IOTEC
#include "HCF_LCD.h" // Vai se tornar HCF_LCD
#include "HCF_ADC.h"   // Vai se tornar HCF_ADC
#include "HCF_MP.h"   // Vai se tornar HCF_MP

// Área das macros
//-----------------------------------------------------------------------------------------------------------------------

#define IN(x) (entradas>>x)&1

// Área de declaração de variáveis e protótipos de funções
//-----------------------------------------------------------------------------------------------------------------------

static const char *TAG = "Placa";
static uint8_t entradas, saidas = 0; // Variáveis de controle de entradas e saídas
static char tecla = '-' ;
char escrever[40];

// Funções auxiliares
//-----------------------------------------------------------------------------------------------------------------------

void abrir_porta(void) {
    ESP_LOGI(TAG, "Abrindo porta...");
    saidas |= (1 << 7); // Sinal de abertura na saída 7
    io_le_escreve(saidas);
    vTaskDelay(pdMS_TO_TICKS(500)); // Pulso de 0,5s
    saidas &= ~(1 << 7); // Fecha a porta
    io_le_escreve(saidas);
    ESP_LOGI(TAG, "Porta fechada.");
}

// Programa Principal
//-----------------------------------------------------------------------------------------------------------------------

void app_main(void)
{
    /////////////////////////////////////////////////////////////////////////////////////   Programa principal
    escrever[39] = '\0';

    // Informações de console
    ESP_LOGI(TAG, "Iniciando...");
    ESP_LOGI(TAG, "Versão do IDF: %s", esp_get_idf_version());

    // Inicializações de periféricos
    iniciar_iotec();      
    entradas = io_le_escreve(saidas); // Limpa as saídas e lê o estado das entradas

    // Inicializar o display LCD
    iniciar_lcd();
    escreve_lcd(1, 0, "Apertar tecla =");  // Instrução para o usuário
    escreve_lcd(2, 0, " Programa Basico");
    
    // Inicializar o componente de leitura de entrada analógica
    esp_err_t init_result = iniciar_adc_CHX(0);
    if (init_result != ESP_OK) {
        ESP_LOGE(TAG, "Erro ao inicializar o componente ADC personalizado");
    }

    // Delay inicial
    vTaskDelay(1000 / portTICK_PERIOD_MS); 
    limpar_lcd();

    // Início do ramo principal                    
    while (1) {                                                                                                                          
        tecla = le_teclado();  // Lê a tecla pressionada
        if (tecla > '0' && tecla <= '8') {
            saidas = 1 << (tecla - '0' - 1);
        } else if (tecla == '9') {
            saidas = 0xFF;  // Liga todas as saídas
        } else if (tecla == '0') {
            saidas = 0x00;  // Desliga todas as saídas
        } else if (tecla == '=') {  // Ação quando tecla "=" for pressionada
            abrir_porta();
        }                                                                                                   
        
        entradas = io_le_escreve(saidas);  // Atualiza as entradas e saídas

        // Atualiza a tela LCD com os estados das entradas
        sprintf(escrever, "INPUTS:%d%d%d%d%d%d%d%d", IN(7), IN(6), IN(5), IN(4), IN(3), IN(2), IN(1), IN(0));
        escreve_lcd(1, 0, escrever);

        // Exibe a tecla pressionada na tela
        sprintf(escrever, "%c", tecla);
        escreve_lcd(2, 14, escrever);
        
        vTaskDelay(10 / portTICK_PERIOD_MS);  // Delay mínimo obrigatório
    }
    
    // Caso erro no programa, desliga o módulo ADC
    adc_limpar();
}
