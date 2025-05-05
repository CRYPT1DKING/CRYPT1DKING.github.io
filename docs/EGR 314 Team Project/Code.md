---
title: Code/Resources
---

## Code

```C
#include "mcc_generated_files/system/system.h"

uint16_t ms=0;
float sec=0;
uint8_t send = 0;
#define Angle_Address_1 0x0E
#define Angle_Address_2 0x0F
#define Slave_Address 0x36
uint8_t  b[2] = {0x0,0x0};
uint8_t angleReg1 = Angle_Address_1;
uint8_t angleReg2 = Angle_Address_2;
uint8_t targetRPM = 0;
uint8_t one = 1;
uint8_t two = 2;
int targetMode = 0;
uint8_t angle = 0;
float angleLast = 0.0f;
int read = 0;

void timer_callback(void)
{
    ms++;
    if (ms>500) 
    {
        ms -= 500;
        if(targetMode == 1)
        {
            send=2;
            read++;
            IO_RC2_Toggle();
        }
        if(targetMode == 0)
        {
            send++;
            read++;
            if(send == 2)
            {
                IO_RC2_Toggle();
            }
        }
    }
}

uint8_t send_message(char * my_message){
    printf("%s",my_message);
    return 1;
}
        

#define BUFSIZE 4
#define MSGSIZE 64
#define TEAMSIZE 5
#define MSGTESTSIZE 64
#define MSGTESTCHAR 0

const char my_id='c';
const char team_ids[TEAMSIZE+1]="abcdX";
char buffer_in[BUFSIZE+1];
char message_in[MSGSIZE+1];
char message_out[MSGSIZE+1];
char c=0;
unsigned int buffer_ii=0;
unsigned int buffer_last_ii = 0;
unsigned int message_ii=0;
unsigned int message_last_ii=0;
unsigned int message_incoming=0;
int messageType2 = 2;
int messageType1 = 1;
uint8_t rpm = 0;
uint16_t raw_angle = 0;
float delta_deg = 0.0;

void fill_string(char * mystring,char value,unsigned int size){
    for (int ii=0;ii<size;ii++){
        mystring[ii]=value;
    }
}

unsigned int find_char(char * mystring, char value,unsigned int size){
    char c=0;
    for (int ii=0;ii<size;ii++){
        c= mystring[ii];
        if (c==value){
            return 1;
        }
    }
    return 0;
}

void handle_message(unsigned int ii){
    message_in[ii] = 0;
    char f = message_in[3];
    if(f == 'c')
    {
        
        if(message_in[4] == 3)
        {
            targetRPM=message_in[5];
            if(targetRPM>0)
            {
                targetMode = 1;
            }
            if(targetRPM==0)
            {
                targetMode = 0;
            }
        }
    }
    else if(f == 'X')
    {
        send_message(message_in);
        return;
        //return broadcast
    }
    else
    {
        send_message(message_in);
        return;
    }
}


int main(void)
{
    buffer_in[BUFSIZE]=0;
    message_in[MSGSIZE]=0;
    fill_string(buffer_in,'a',BUFSIZE);
    fill_string(message_in,'_',MSGSIZE);
    message_in[MSGTESTSIZE]=MSGTESTCHAR;
    
    uint16_t ms_last=0;
    uint16_t sec_last=0;
    uint16_t adc_value;
    float voltage;
    SYSTEM_Initialize();
    INTERRUPT_GlobalInterruptEnable(); 
    INTERRUPT_PeripheralInterruptEnable(); 
    Timer1_Initialize();
    Timer1_Start();

    TMR1_OverflowCallbackRegister(timer_callback);

    UART1_Initialize();
    while(1)
    {
        if(EUSART1_IsRxReady())
        {
            c= EUSART1_Read();
            buffer_in[buffer_ii]=c;
            if (buffer_in[buffer_last_ii]=='A' & buffer_in[buffer_ii]=='Z'){
                fill_string(message_in,'_',MSGSIZE);
                message_in[MSGTESTSIZE]=MSGTESTCHAR;
                message_incoming=1;
                message_in[0] = buffer_in[buffer_last_ii];
                message_ii=1;
            }
            if (buffer_in[buffer_last_ii]=='Y' & buffer_in[buffer_ii]=='B'){
                message_incoming=0;
                message_in[message_ii] = buffer_in[buffer_ii];
                message_last_ii= message_ii;
                message_ii = message_ii+1;
                handle_message(message_ii);
            }
            if (message_incoming!=0){
                message_in[message_ii] = buffer_in[buffer_ii];

                if (message_ii==2){
                    unsigned result = 0;
                    char d=0;
                    d = message_in[message_ii];
                    result= find_char(team_ids,d,TEAMSIZE);
                    if (result==0){
                        //printf("AZbaPIC: sender not in teamYB\n");
                        message_incoming = 0;
                        message_ii=0;
                    } else {
                    }
                }
                
                if (message_ii==2){
                    unsigned result = 0;
                    char d=0;
                    d = message_in[message_ii];
                    if (d=='c'){
                        //printf("AZbaPIC: sender is yourselfYB\n");
                        message_incoming = 0;
                        message_ii=0;
                    } else {
                    }
                }

                if (message_ii==3){
                    unsigned result = 0;
                    char d=0;
                    d = message_in[message_ii];
                    result= find_char(team_ids,d,TEAMSIZE);
                    if (result==0){ 
                        //printf("AZbaPIC: receiver not in teamYB\n");
                        message_incoming = 0;
                        message_ii=0;
                    } else {
                    }
                }

                message_last_ii= message_ii;
                message_ii = message_ii+1;
                if (message_ii<MSGTESTSIZE){} else{
                    //printf("AZbaPIC: message too large. deletingYB\n");
                    message_incoming=0;
                    message_ii=0;
                }
            }
            buffer_last_ii= buffer_ii;
            buffer_ii = (buffer_ii+1)%BUFSIZE;
        }
        sec_last = sec;
        ms_last = ms;
        if (read == 1)
        {
            angleLast = angle;

            i2c_WriteRead(Slave_Address, &angleReg1, 1, b, 1);
            __delay_ms(10); 
            i2c_WriteRead(Slave_Address, &angleReg2, 1, &b[1], 1);

            uint16_t raw_angle = ((b[0] & 0x0F) << 8) | b[1];
            angle = ((float)raw_angle * 360.0f) / 4096.0f;

            // Handle angle wraparound
            if (angle - angleLast > 180.0f) {
                delta_deg = angle - 360.0f - angleLast;
            } else if (angle - angleLast < -180.0f) {
                delta_deg = 360.0f + angle - angleLast;
            } else {
                delta_deg = angle - angleLast;
            }

            // Check if the angle change is too small for RPM calculation
            if (delta_deg < 0.1f && delta_deg > -0.1f) {
                rpm = 0;  // Set RPM to 0 if the change is too small
            } else {
                rpm = (delta_deg / 360.0f) * 60.0f;
            }

            // Clamp RPM to uint8_t range [0, 255]
            if (rpm < 0) {
                rpm = 0;  // Clamp to 0 if RPM is negative
            }
            if (rpm > 255) {
                rpm = 255;  // Clamp to 255 if RPM is greater than 255
            }

            // Cast to uint8_t
            rpm = (uint8_t)rpm;

            read = 0;
        }
        if(send == 2)
        {
            printf("AZcX");
            __delay_ms(5);
            EUSART1_Write(messageType2);
            __delay_ms(5);
            EUSART1_Write(rpm);
            __delay_ms(5);
            printf("YB");
            __delay_ms(5);
            
            if(targetMode == 1)
            {
                if(rpm<targetRPM)
                {
                    printf("AZcX");
                    __delay_ms(5);
                    EUSART1_Write(messageType1);
                    __delay_ms(5);
                    EUSART1_Write(one);
                    __delay_ms(5);
                    printf("YB");
                    __delay_ms(5);
                }
                if(rpm>targetRPM)
                {
                    printf("AZcX");
                    __delay_ms(5);
                    EUSART1_Write(messageType1);
                    __delay_ms(5);
                    EUSART1_Write(two);
                    __delay_ms(5);
                    printf("YB");
                    __delay_ms(5);
                }
            }
            send = 0;
        }
    }
}
```

* [MPLabX Files](Messaging.X.zip)