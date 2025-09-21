 
#include "hal_data.h" 
#include <string.h> 
#include <stdio.h> 
#include <stdlib.h> 
 
#define LED_PIN     BSP_IO_PORT_01_PIN_04 
#define BUTTON_PIN  BSP_IO_PORT_01_PIN_05 
#define CONTROL_PIN BSP_IO_PORT_04_PIN_08  // Pin 408 
 
// Buffer settings 
#define GPS_BUFFER_SIZE 256 
#define SMS_BUFFER_SIZE 300 
 
// GPS data buffer 
char gps_rx_buffer[GPS_BUFFER_SIZE]; 
volatile uint16_t gps_rx_index = 0; 
volatile bool gps_data_ready = false; 
 
// GPS coordinates storage - INITIALIZE TO ZERO FOR LIVE DATA 
int32_t latitude_int = 0;    // Will be updated with live GPS data 
int32_t longitude_int = 0;   // Will be updated with live GPS data 
bool gps_fix_valid = false;  // Will be set true when valid GPS received 
 
// Debug counters 
volatile uint32_t gps_message_count = 0; 
volatile uint32_t parse_attempt_count = 0; 
 
// Button press counter 
volatile uint8_t button_press_count = 0; 
#define REQUIRED_BUTTON_PRESSES 3 
 
// SMS phone number 
const char* sms_phone_number = "+91 0000000000"; 
 
// GPS UART1 callback function 
void gps_uart_callback(uart_callback_args_t *p_args) 
{ 
    if (p_args->event == UART_EVENT_RX_CHAR) 
    { 
        char received_char = (char)p_args->data; 
 
        if (gps_rx_index < (GPS_BUFFER_SIZE - 1)) 
        { 
            gps_rx_buffer[gps_rx_index++] = received_char; 
 
            if (received_char == '\n') 
            { 
                gps_rx_buffer[gps_rx_index] = '\0'; 
                gps_data_ready = true; 
                gps_message_count++; 
            } 
        } 
        else 
        { 
            gps_rx_index = 0; 
        } 
    } 
} 
 
// Enhanced GPS parsing for ANY latitude/longitude values (not just hardcoded) 
void parse_gps_coordinates(char* nmea_sentence) 
{ 
    parse_attempt_count++; 
 
    // Look for ANY latitude pattern (not just "1221") 
    // Pattern: DDMM.MMMMM,N,DDDMM.MMMMM,E 
    char *pos = nmea_sentence; 
 
    // Find any sequence that looks like: digit digit digit digit . digit digit digit 
    while (*pos) 
    { 
        // Look for 4 digits followed by a decimal point (latitude pattern) 
        if (*pos >= '0' && *pos <= '9' && 
            *(pos+1) >= '0' && *(pos+1) <= '9' && 
            *(pos+2) >= '0' && *(pos+2) <= '9' && 
            *(pos+3) >= '0' && *(pos+3) <= '9' && 
            *(pos+4) == '.') 
        { 
            // Found potential latitude, extract it 
            char lat_str[15] = {0}; 
            char lon_str[15] = {0}; 
 
            // Extract latitude 
            int i = 0; 
            char *extract_pos = pos; 
            while (*extract_pos && *extract_pos != ',' && i < 12) 
            { 
                lat_str[i++] = *extract_pos++; 
            } 
            lat_str[i] = '\0'; 
 
            // Check if followed by ,N, or ,S, 
            if (*extract_pos == ',' && (*(extract_pos+1) == 'N' || *(extract_pos+1) == 'S')) 
            { 
                char lat_dir = *(extract_pos+1); 
                extract_pos += 3; // Skip ",N," or ",S," 
 
                // Extract longitude (should be 5 digits + decimal) 
                i = 0; 
                while (*extract_pos && *extract_pos != ',' && i < 12) 
                { 
                    lon_str[i++] = *extract_pos++; 
                } 
                lon_str[i] = '\0'; 
 
                // Check if followed by ,E, or ,W, 
                if (*extract_pos == ',' && (*(extract_pos+1) == 'E' || *(extract_pos+1) == 'W')) 
                { 
                    char lon_dir = *(extract_pos+1); 
 
                    // Convert coordinates if we have valid strings 
                    if (strlen(lat_str) >= 7 && strlen(lon_str) >= 8) 
                    { 
                        // Latitude conversion (DDMM.MMMMM to decimal degrees) 
                        int lat_deg = (lat_str[0] - '0') * 10 + (lat_str[1] - '0'); 
                        float lat_min = atof(&lat_str[2]); 
                        float lat_decimal = lat_deg + (lat_min / 60.0); 
                        if (lat_dir == 'S') lat_decimal = -lat_decimal; 
                        latitude_int = (int32_t)(lat_decimal * 1000000); 
 
                        // Longitude conversion (DDDMM.MMMMM to decimal degrees) 
                        int lon_deg = (lon_str[0] - '0') * 100 + (lon_str[1] - '0') * 10 + (lon_str[2] - '0'); 
                        float lon_min = atof(&lon_str[3]); 
                        float lon_decimal = lon_deg + (lon_min / 60.0); 
                        if (lon_dir == 'W') lon_decimal = -lon_decimal; 
                        longitude_int = (int32_t)(lon_decimal * 1000000); 
 
                        gps_fix_valid = true; 
                        return; // Successfully parsed live coordinates 
                    } 
                } 
            } 
        } 
        pos++; 
    } 
} 
 
// Send SMS with LIVE GPS coordinates 
void send_gps_sms(void) 
{ 
    char sms_message[SMS_BUFFER_SIZE]; 
 
    if (gps_fix_valid && (latitude_int != 0 || longitude_int != 0)) 
    { 
        // Convert integer coordinates back to decimal format for display 
        int32_t lat_whole = latitude_int / 1000000; 
        int32_t lat_frac = (latitude_int < 0) ? (-latitude_int % 1000000) : (latitude_int % 1000000); 
        int32_t lon_whole = longitude_int / 1000000; 
        int32_t lon_frac = (longitude_int < 0) ? (-longitude_int % 1000000) : (longitude_int % 1000000); 
 
        // Handle negative coordinates properly 
        if (latitude_int < 0 && lat_whole == 0) lat_whole = 0; // For -0.xxxxx case 
        if (longitude_int < 0 && lon_whole == 0) lon_whole = 0; // For -0.xxxxx case 
 
        // Build SMS message with LIVE coordinates 
        snprintf(sms_message, SMS_BUFFER_SIZE, 
                "LIVE GPS Location:\nLat: %s%ld.%06ld\nLon: %s%ld.%06ld\nMsgs: %lu Parses: 
%lu\nGoogle Maps: https://maps.google.com/?q=%ld.%06ld,%ld.%06ld\r\n", 
                (latitude_int < 0) ? "-" : "", (lat_whole < 0) ? -lat_whole : lat_whole, lat_frac, 
                (longitude_int < 0) ? "-" : "", (lon_whole < 0) ? -lon_whole : lon_whole, lon_frac, 
                gps_message_count, parse_attempt_count, 
                lat_whole, lat_frac, lon_whole, lon_frac); 
    } 
    else 
    { 
        snprintf(sms_message, SMS_BUFFER_SIZE, 
                "GPS Status: No valid GPS fix yet.\nMessages received: %lu\nParse attempts: %lu\nWaiting 
for GPS signal...\r\n", 
                gps_message_count, parse_attempt_count); 
    } 
 
    // Send SMS 
    char cmgf_cmd[] = "AT+CMGF=1\r\n"; 
    R_SCI_UART_Write(&g_uart0_ctrl, cmgf_cmd, strlen(cmgf_cmd)); 
    R_BSP_SoftwareDelay(1000, BSP_DELAY_UNITS_MILLISECONDS); 
 
    char cmgs_cmd[50]; 
    snprintf(cmgs_cmd, sizeof(cmgs_cmd), "AT+CMGS=\"%s\"\r\n", sms_phone_number); 
    R_SCI_UART_Write(&g_uart0_ctrl, cmgs_cmd, strlen(cmgs_cmd)); 
    R_BSP_SoftwareDelay(1000, BSP_DELAY_UNITS_MILLISECONDS); 
 
    R_SCI_UART_Write(&g_uart0_ctrl, sms_message, strlen(sms_message)); 
    R_BSP_SoftwareDelay(500, BSP_DELAY_UNITS_MILLISECONDS); 
 
    char ctrl_z = 0x1A; 
    R_SCI_UART_Write(&g_uart0_ctrl, &ctrl_z, 1); 
    R_BSP_SoftwareDelay(5000, BSP_DELAY_UNITS_MILLISECONDS); 
} 
 
void hal_entry(void) 
{ 
    fsp_err_t err; 
 
    // Configure pins 
    R_IOPORT_PinCfg(&g_ioport_ctrl, LED_PIN, IOPORT_CFG_PORT_DIRECTION_OUTPUT); 
    R_IOPORT_PinCfg(&g_ioport_ctrl, CONTROL_PIN, IOPORT_CFG_PORT_DIRECTION_OUTPUT); 
 
    // Initialize control pin to LOW 
    R_IOPORT_PinWrite(&g_ioport_ctrl, CONTROL_PIN, BSP_IO_LEVEL_LOW); 
 
    R_SCI_UART_Open(&g_uart0_ctrl, &g_uart0_cfg); 
 
    err = R_SCI_UART_Open(&g_uart1_ctrl, &g_uart1_cfg); 
    if (err != FSP_SUCCESS) 
    { 
        while(1) 
        { 
            R_IOPORT_PinWrite(&g_ioport_ctrl, LED_PIN, BSP_IO_LEVEL_HIGH); 
            R_BSP_SoftwareDelay(100, BSP_DELAY_UNITS_MILLISECONDS); 
            R_IOPORT_PinWrite(&g_ioport_ctrl, LED_PIN, BSP_IO_LEVEL_LOW); 
            R_BSP_SoftwareDelay(100, BSP_DELAY_UNITS_MILLISECONDS); 
        } 
    } 
 
    R_SCI_UART_CallbackSet(&g_uart1_ctrl, gps_uart_callback, NULL, NULL); 
 
    bsp_io_level_t button_state, last_state = BSP_IO_LEVEL_HIGH; 
 
    while (1) 
    { 
        if (gps_data_ready) 
        { 
            parse_gps_coordinates(gps_rx_buffer); // Parse LIVE GPS data 
            gps_rx_index = 0; 
            gps_data_ready = false; 
 
            R_IOPORT_PinWrite(&g_ioport_ctrl, LED_PIN, BSP_IO_LEVEL_HIGH); 
            R_BSP_SoftwareDelay(50, BSP_DELAY_UNITS_MILLISECONDS); 
            R_IOPORT_PinWrite(&g_ioport_ctrl, LED_PIN, BSP_IO_LEVEL_LOW); 
        } 
 
        R_IOPORT_PinRead(&g_ioport_ctrl, BUTTON_PIN, &button_state); 
 
        // Detect button press (rising edge) 
        if (button_state == BSP_IO_LEVEL_HIGH && last_state == BSP_IO_LEVEL_LOW) 
        { 
            button_press_count++; 
 
            // Flash LED to indicate button press 
            R_IOPORT_PinWrite(&g_ioport_ctrl, LED_PIN, BSP_IO_LEVEL_HIGH); 
            R_BSP_SoftwareDelay(200, BSP_DELAY_UNITS_MILLISECONDS); 
            R_IOPORT_PinWrite(&g_ioport_ctrl, LED_PIN, BSP_IO_LEVEL_LOW); 
 
            // Check if we've reached the required number of presses 
            if (button_press_count >= REQUIRED_BUTTON_PRESSES) 
            { 
                // Set control pin HIGH when counter reaches 3 
                R_IOPORT_PinWrite(&g_ioport_ctrl, CONTROL_PIN, BSP_IO_LEVEL_HIGH); 
 
                // Keep LED on while sending SMS 
                R_IOPORT_PinWrite(&g_ioport_ctrl, LED_PIN, BSP_IO_LEVEL_HIGH); 
                send_gps_sms(); // Send current LIVE GPS coordinates 
 
                // Set control pin LOW after SMS is sent 
                R_IOPORT_PinWrite(&g_ioport_ctrl, CONTROL_PIN, BSP_IO_LEVEL_LOW); 
 
                button_press_count = 0; // Reset counter 
                R_BSP_SoftwareDelay(1000, BSP_DELAY_UNITS_MILLISECONDS); 
                R_IOPORT_PinWrite(&g_ioport_ctrl, LED_PIN, BSP_IO_LEVEL_LOW); 
            } 
        } 
 
        last_state = button_state; 
        R_BSP_SoftwareDelay(10, BSP_DELAY_UNITS_MILLISECONDS); 
    } 
 
#if BSP_TZ_SECURE_BUILD 
    R_BSP_NonSecureEnter(); 
#endif 
} 
 
void R_BSP_WarmStart(bsp_warm_start_event_t event) 
{ 
    if (BSP_WARM_START_RESET == event) 
    { 
#if BSP_FEATURE_FLASH_LP_VERSION != 0 
        R_FACI_LP->DFLCTL = 1U; 
#endif 
    } 
 
    if (BSP_WARM_START_POST_C == event) 
    { 
        R_IOPORT_Open(&IOPORT_CFG_CTRL, &IOPORT_CFG_NAME); 
#if BSP_CFG_SDRAM_ENABLED 
        R_BSP_SdramInit(true); 
#endif 
} 
} 
#if BSP_TZ_SECURE_BUILD 
FSP_CPP_HEADER 
BSP_CMSE_NONSECURE_ENTRY void template_nonsecure_callable(); 
BSP_CMSE_NONSECURE_ENTRY void template_nonsecure_callable() {} 
FSP_CPP_FOOTER 
#endif

