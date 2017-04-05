#Temperature Sensor, Heartbeat Sensor and GSM:
#include <htc.h> 
#include <pic.h>
#define _XTAL_FREQ 8000000
#define	RS RC0
#define	RW RC1
#define EN RC2
#define key RB0
#define indication RC5
#define fan PORTBbits.RB7 

void TransByte(unsigned char Byte);
void at_cmd_without_end( const unsigned char *cmnd,unsigned int lnth);
void sel_text_mode(void);
void send_at_command(const unsigned char *cmnd,unsigned int lnth);
void send_sms( const unsigned char *mono,const unsigned char *sms, unsigned int len);
void GSMinit(void);
void adc_init();
void lcd_init();
void lcd_cmd(unsigned char a);
void lcd_data(unsigned char b);
hex_to_ascii(unsigned int check);
void lcd_string(char *p);
void dis_num(unsigned int data, unsigned char digit);
void read_heartbeats();

void main()
	{
		unsigned int check=0,j,avg=0;
    	TRISA=0xFF;
		TRISB=0xFF;
		PORTB=0x00;
		TRISC=0x00;
		TRISD=0x00;
		
		TXSTA=0x26;			//TX enable & High baud rate & TX shift register status bit  	
		RCSTA=0x90;			//Serial enable and Continuous reception enble
		SPBRG=51;			// 9600 Baud Rate
		
		lcd_init();
        lcd_cmd(0x80);
		lcd_string("Fireman Safety");
		lcd_cmd(0xC0);
		lcd_string("    System");
        //__delay_ms(2000);
		//lcd_cmd(0x01);
		
		GSMinit();			// GSM module initialisation

		adc_init();
		indication = 0;
		while(1)
		{	
		
           if(key == 0)
		   { 
			indication = 1;
			__delay_ms(50);
			GO=1;			//ADC Go bit	
			__delay_ms(50);
	
		
			for(j=0;j<50;j++)
			{	
			check = check + ((ADRESH<<8) + ADRESL); //10 bit adc result
			__delay_ms(50);
              }

			avg=check/50;

				hex_to_ascii(avg);
				check=0;
			//lcd_cmd(0x01);
			read_heartbeats();

			__delay_ms(1000);

			sel_text_mode();
			__delay_ms(1000);
			send_sms("4699010373","Emergency:",9);
		  }
		
	 	else
			{
				//lcd_cmd(0x01);
				indication = 0;
				lcd_cmd(0x80);
				lcd_string("Fireman Safety  ");
				lcd_cmd(0xC0);
				lcd_string("    System");
			}
						
		}	
	}

/////////////////////////////ADC////////////////////

void adc_init()
	{
			
		ADCON0=0X80;
		ADCON1=0XCE;
		ADON=1;
	    
		GO=1;

		while(GO);
		
	}

/////////////////LCD///////////////////////////////

	void lcd_init()
	{
		
	unsigned char a[]={0x02,0x28,0x0C,0x01,0x06,0x80};
	unsigned char i;

			for(i=0;i<6;i++)
			{
				lcd_cmd(a[i]);
				__delay_ms(50);	
			}	

	}

	void lcd_cmd(unsigned char a)
	{

		PORTD=(a & 0xF0);

		RS=0;
		RW=0;
		EN=1;
		__delay_ms(10);
		EN=0;
	
		PORTD=((a<<4) & (0xF0) );
		EN=1;
		__delay_ms(10);
		EN=0;
			
	}
	
	void lcd_data(unsigned char b)
		{			
			RS=1;
			RW=0;
	PORTD=(b & 0xF0);
		
			EN=1;
			__delay_ms(1);
			EN=0;
				
			PORTD=((b<<4) & (0xF0) );
				
			EN=1;
			__delay_ms(1);
			EN=0;
	
		}	


		hex_to_ascii(unsigned int check)
		{
			unsigned int temp;
			unsigned char a[4],i;
			
			lcd_cmd(0x01);
	
			temp=check*0x05;
			a[2]=temp%10;
			temp=temp/10;

			a[1]=temp%10;
			temp=temp/10;
			
			a[0]=temp%10;
				
    		
			lcd_cmd(0x80);
            lcd_string("Temp:");
			lcd_cmd(0x85);
				
			for(i=0;i<2;i++)		
			lcd_data(a[i]+0x30);
				
			lcd_cmd(0x87);
		    lcd_data('.');
			
			lcd_cmd(0x88);
			lcd_data(a[2]+0x30);
	    
			lcd_cmd(0x89);
		    lcd_data(223);

			lcd_cmd(0x8A);
		    lcd_data('C');

        }
	
void lcd_string(char *p)
{
while(*p!='\0')
{
lcd_data(*p);
*p++;
}
}

////////////////////////////GSM///////////////////////

void TransByte(unsigned char Byte)		// GSM serial data transmission
{
	while(!TXIF) ;						// Wait fot TX flag	
	TXREG = Byte ;						// Copy the contents to TX register 
}

void at_cmd_without_end( const unsigned char *cmnd,unsigned int lnth)
 {
    unsigned int i;
    for(i=0;i<lnth;i++)
	 {
	    TransByte(*cmnd);				//Sending of AT commands using pointer
		*cmnd++;
	 }
	 
  }


void sel_text_mode(void)
{
	send_at_command("AT+CMGF=1\r",11);
		__delay_ms(1000);
}

void send_at_command(const unsigned char *cmnd,unsigned int lnth)
 {
    unsigned int i;
    for(i=0;i<lnth;i++)
	 {
	    TransByte(*cmnd);
		cmnd++;
	 }
 }

void send_sms( const unsigned char *mono,const unsigned char *sms, unsigned int len)
{
   	__delay_ms(1000);
	at_cmd_without_end("AT+CMGS=",8);
	TransByte('"');
	at_cmd_without_end(mono,10);
	TransByte('"');
	send_at_command("\r",2);
	__delay_ms(2000);
  
    at_cmd_without_end(sms,len);
	TransByte(0x1a);
	send_at_command("\r",2);
	__delay_ms(1000);
}

void GSMinit(void)
{
    send_at_command("AT\r",4);
    	__delay_ms(1000);    
}    


////////////////Heart beats////////////////

void read_heartbeats()
{
	unsigned int hb=1;
	while(RB2);
	while(!RB2)
    {
   		hb++;
		__delay_ms(1);
	}
	
	hb=30000/hb;
	lcd_cmd(0xC0);
	lcd_string("HBR=   bpm");
	lcd_cmd(0xC4);
	dis_num(hb,3);
}

void dis_num(unsigned int data, unsigned char digit)
{
	if(digit>3)
	lcd_data('0'+(data/1000)%10);
	if(digit>2)
	lcd_data('0'+(data/100)%10);
	if(digit>1)
	lcd_data('0'+(data/10)%10);
	if(digit>0)
	lcd_data('0'+(data/1)%10);
}


7.2 Coding for GPS:
Since it was difficult to implement the simulations of GPS without hardware, I have implemented this part separately. Here is the code for the GPS:

#include <htc.h> 
#include <pic.h>
#define _XTAL_FREQ 8000000

#define	RS RC0
#define	RW RC1
#define EN RC2

void lcd_init();
void lcd_cmd(unsigned char a);
void lcd_data(unsigned char b);
void lcd_string(char *p);

void main()
{
	unsigned char i=0,j;
	unsigned char rcev[80];
	TRISA=0xFF;
	TRISB=0xFF;
	PORTB=0x00;
	TRISC=0xF0;
	TRISD=0x00;
	
	lcd_init();
	
	TXSTA=0x26;		//TX enable & High baud rate & TX shift register //status bit  	
	RCSTA=0x90;		//Serial enable and Continuous reception enble
	SPBRG=51;			// 9600 Baud Rate
	
	
//	__delay_ms(1000);

	while(1)
	{
			//send_data("$GPRMC,064951.000,A,2307.1256,N,12016.4438,E,0.03,165.48,111215, 3.05,W,A*55\r",78);
			lcd_cmd(0x80);
			lcd_string("Fireman Location");
			while(!RCIF);
			while(RCREG!='$');
			TXREG = RCREG;
			for(i=0;i<75;i++)
			{
				while(!RCIF);
            	rcev[i]=RCREG;
				TXREG=rcev[i];
				while(!TXIF) ;
				__delay_ms(100);
			}
			lcd_cmd(0xC0);
			for(j=0;j<7;j++)
			lcd_data(rcev[19+j]);
			lcd_data(rcev[29]);
			for(j=0;j<7;j++)
			lcd_data(rcev[31+j]);
			lcd_data(rcev[42]);
	}	
}

/////////////////LCD///////////////////////////////

	void lcd_init()
	{
		
		unsigned char a[]={0x02,0x28,0x0C,0x01,0x06,0x80};
		unsigned char i;

			for(i=0;i<6;i++)
			{
				lcd_cmd(a[i]);
				__delay_ms(50);	
			}	

	}

	void lcd_cmd(unsigned char a)
	{

		PORTD=(a & 0xF0);

		RS=0;
		RW=0;
		EN=1;
		__delay_ms(10);
		EN=0;
	
		PORTD=((a<<4) & (0xF0) );
		EN=1;
		__delay_ms(10);
		EN=0;
			
	}
	
	void lcd_data(unsigned char b)
		{			
			RS=1;
			RW=0;
			PORTD=(b & 0xF0);
		
			EN=1;
			__delay_ms(1);
			EN=0;
				
			PORTD=((b<<4) & (0xF0) );
				
			EN=1;
			__delay_ms(1);
			EN=0;
	
		}	

void lcd_string(char *p)
{
while(*p!='\0')
{
lcd_data(*p);
*p++;
}
}
