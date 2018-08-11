# Embedded-System
Railway gate crossing control using Embedded System programming in C.

#include<reg52.h>

sbit IR=P1^5;

sbit RLY1=P1^0;
sbit RLY2=P1^1;

sbit track=P1^3;
sbit motor=P1^4;
sbit buzz=P1^2;


sbit RS=P3^7;
sbit EN=P3^6;

//---------------------------------------
// Forward function declaration
//---------------------------------------
void send_cmd(unsigned char *cmd);
void Txmsg(unsigned char *no,unsigned char *msg);
unsigned char Rxmsg(void);
void lcdinit(void);
void lcdclear(void);
void lcdData(unsigned char l);
void lcdcmd(unsigned char k);
void DelayMs(unsigned int count);
void DelayUs(unsigned int count);
void InitModem(void);
void lcd_puts(unsigned char *p);
void uart_puts(unsigned char *p);
void uart_gets(unsigned char *p,unsigned char len);
void uart_putchar(unsigned char p);
void uart_getchar(unsigned char *p,unsigned char ch);

unsigned char status=0,ret=0,mno[]="+918860119661";
bit ir_stat=0;
//---------------------------------------
// Main rotine
//---------------------------------------
void main()
{
	unsigned char a=0,b=0,c=0,d=0;
	TMOD=0x20;			                        // Configure UART at 9600 baud rate
	TH1=0xFD;
	SCON=0x50;
	TR1=1;
	
	IR=1;
	track=1;
	motor=1;
	buzz=0;
	RLY1=0;
	RLY2=0;

lcdinit();			                        // Initialize LCD
lcd_puts("INITIALIZING    MODEM...PLZ WAIT");
DelayMs(10000); 
InitModem();		// Initialize Modem
//Txmsg(mno,"GSM BASED HOME APPLIANCE CONTROL");
DelayMs(1000);

while(1)
{
	ret=Rxmsg();
		
		if(ret==1)
		{
			if(motor==0)
			{
				buzz=1;
				DelayMs(10000);
				buzz=0;
				DelayMs(5000);
				if(motor==0)
				{
					ir_stat=1;
					RLY1=0;
					RLY2=1;
					DelayMs(1500);
					RLY1=0;
					RLY2=0;
					Txmsg(mno,"Gate Closed");
					lcdclear();
					lcd_puts("Gate Closed");
				}
				else
				{
					Txmsg(mno,"Gate motor is not working.");
				}
			}
		}
		if(ir_stat==1)
		{
			ir_stat=0;
			while(IR==0);
			while(IR==1);
			DelayMs(2000);
			RLY1=1;
			RLY2=0;
			DelayMs(1500);
			RLY1=0;
			RLY2=0;
			Txmsg(mno,"Gate Open");
			lcdclear();
			lcd_puts("Gate Open");
			DelayMs(1000);
		}
		if(track==1)
		{
			if(a==0){
				a=1;
				Txmsg(mno,"Track disjoint.");
				lcdclear();
				lcd_puts("Track disjoint.");
				DelayMs(1000);
			}
		}
		else
			a=0;
		if(motor==1)
		{
			if(b==0){
				b=1;
				buzz=1;
				Txmsg(mno,"Gate motor is not working.");
				lcdclear();
				lcd_puts("Gate motor is not working.");
				DelayMs(1000);
				buzz=0;
			}
		}
		else
			b=0;
		

} 
}

//----------------------------------------------------
// send command subroutine to check the connectivity of modem
//----------------------------------------------------
void send_cmd(unsigned char *cmd)
{
unsigned char i,buff[6];
retry:
lcdclear();
uart_puts(cmd);		     // Sending Command
uart_putchar(0x0d);      // Enter
uart_gets(buff,6);		 // Receive response
DelayMs(100);

for(i=0;i<6;i++)	     // Compare response
{
if(buff[i]=='E' || buff[i]=='R')
goto retry;
if(buff[i]=='O' || buff[i]=='K')
return;
}
goto retry;
}

//---------------------------------------
// Modem initialization subroutine
//---------------------------------------
void InitModem(void)
{
send_cmd("AT");					  // AT command sending to check the connectivity
send_cmd("ATE0");				  // ATE0 sending to turn off the echo 				      
send_cmd("AT+CMGF=1");		      // sending AT+CMGF  to set the text mode
send_cmd("AT+CMGD=1");			  // sending AT+CMGD to delete message 
}

void Txmsg(unsigned char *no, unsigned char *msg)
{
	unsigned char i,buff[20];

	retry:
	send_cmd("AT");					  // AT command sending to check the connectivity
	lcdclear();
	uart_puts("AT+CMGS=\"");
	uart_puts(no);
	uart_putchar('"');
	uart_putchar(0x0d); 
	lcdclear();
	uart_getchar(buff,'>');
	uart_puts(msg);
	uart_putchar(26);
	lcdclear();
	uart_gets(buff,20);
	DelayMs(1000);

	for(i=0;i<10;i++)			     //command to recv data
	{
	if(buff[i]=='+' || buff[i+1]=='C' || buff[i+2]=='M' || buff[i+3]=='G')
	return;
	}
	goto retry;
} 

//---------------------------------------
// Recieve message subroutine
//---------------------------------------
unsigned char Rxmsg(void)
{
	unsigned char i=0,ret=0;
	unsigned int j=0;
	unsigned char idata c[84];	

	send_cmd("AT");					  // AT command sending to check the connectivity
	DelayMs(100);
	lcdclear();
	uart_puts("AT+CMGR=1");
	DelayMs(100);
	uart_putchar(0x0d); 

	lcdcmd(0xC0);

	for(i=0;i<84;i++)
	{
	j=0;
	while(RI==0)
	{
	if(j>=10000)
	goto timeout;
	DelayUs(1);
	j++;
	}
	c[i]=SBUF;
	RI=0;
	lcdData(c[i]);
	}
	DelayMs(1000);

	timeout:
	for(i=0;i<5;i++)			  //command to recv data
	{
	if((c[i]=='O') || (c[i]=='K'))
	return ret;
	}

	for(i=0;i<84;i++)
	{
	if((c[i]=='U') && (c[i+1]=='N') && (c[i+2]=='R') && (c[i+3]=='E') && (c[i+4]=='A') && (c[i+5]=='D'))
	goto sucess1;
	}
	goto delete;

	sucess1:
	i=i+9;

	for(j=0;j<13;j++)
	{
	mno[j]=c[i];
	i++;
	}
	mno[j]='\0';

	for(i=0;i<84;i++)
	{
	if((c[i]=='3') && (c[i+1]=='5') && (c[i+2]=='7') && (c[i+3]=='9') && (c[i+4]=='1'))	                                                                                   
	goto sucess;
	}
	goto delete;

	sucess:
	for(i=0;i<84;i++)
	{
		if((c[i]=='C') && (c[i+1]=='L') && (c[i+2]=='O')&& (c[i+3]=='S') && (c[i+4]=='E'))
		{
			ret=1;
		}

		
	}
	delete:
	send_cmd("AT+CMGD=1");			  // sending AT+CMGD to delete message 
	return ret;
}

//---------------------------------------
// Lcd initialization subroutine
//---------------------------------------
void lcdinit(void)
{
lcdcmd(0x38);
DelayMs(25);
lcdcmd(0x0E);
DelayMs(25);
lcdcmd(0x01);
DelayMs(25);
lcdcmd(0x06);
DelayMs(25);
lcdcmd(0x80);
DelayMs(25);
}

//---------------------------------------
// Lcd initialization subroutine
//---------------------------------------
void lcdclear(void)
{
	lcdcmd(0x01);
	DelayMs(10);
	lcdcmd(0x80);
	DelayMs(10);
}

//---------------------------------------
// Lcd data display
//---------------------------------------
void lcdData(unsigned char l)
{
	P2=l;
	RS=1;
	EN=1;
	DelayMs(1);
	EN=0;
}

//---------------------------------------
// Lcd command
//---------------------------------------
void lcdcmd(unsigned char k)
{
	P2=k;
	RS=0;
	EN=1;
	DelayMs(1);
	EN=0;
}			   

//---------------------------------------
// Delay mS function
//---------------------------------------
void DelayMs(unsigned int count) 
{											// mSec Delay 11.0592 Mhz 
    unsigned int i;		      				// Keil v7.5a 
    while(count) {
        i = 115; 			 				// 115	exact value
		while(i>0) i--;
        count--;
    }
}

//---------------------------------------
// Delay mS function
//---------------------------------------
void DelayUs(unsigned int count) 
{											// mSec Delay 11.0592 Mhz 
    unsigned int i;		      				// Keil v7.5a 
    while(count) {
        i = 10; 			 				   // 115	exact value
		while(i>0) i--;
        count--;
    }
}

void lcd_puts(unsigned char *p)
{
	unsigned char i=0;

	while(p[i]!='\0')
	{
		lcdData(p[i]);
		i++;
		if(i==16)
		lcdcmd(0xC0);
		
	}
}

void uart_puts(unsigned char *p)
{
while(*p!='\0')
{
lcdData(*p);
uart_putchar(*p);
DelayMs(10);
p++;
}
}

void uart_putchar(unsigned char p)
{
SBUF=p;
while(TI==0);
TI=0;
}

void uart_gets(unsigned char *p,unsigned char len)
{
unsigned int i,j;

for(i=0;i<len;i++)			  //command to recv data
{
j=0;
while(RI==0)
{
if(j>=10000)
break;
DelayUs(1);
j++;
}
p[i]=SBUF;
RI=0;
lcdData(p[i]);
}
}

void uart_getchar(unsigned char *p,unsigned char ch)
{
unsigned int i,j;

i=0;
do
{
j=0;
while(RI==0)
{
if(j>=10000)
break;
DelayUs(1);
j++;
}
p[i]=SBUF;
RI=0;
lcdData(p[i]);
i++;
} while(p[i]!=ch && i<10);
}
