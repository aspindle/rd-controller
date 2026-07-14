//Copyright (c) 2011-2020 <>< Charles Lohr - Under the MIT/x11 or NewBSD License you choose.
// NO WARRANTY! NO GUARANTEE OF SUPPORT! USE AT YOUR OWN RISK

#include <stdio.h>
#include <stdlib.h>
#include <math.h>
#include <float.h>
#include <string.h>
#include "os_generic.h"
#include <asset_manager.h>
#include <asset_manager_jni.h>
#include <android_native_app_glue.h>
#include <android/sensor.h>
#include "CNFGAndroid.h"
#include <android/log.h>
#include <arpa/inet.h>
#include <sys/socket.h>
#include <unistd.h>
#include <errno.h>


#define CNFG_IMPLEMENTATION
#define CNFG3D

#include "CNFG.h"
#include <GLES2/gl2.h>


#define LOG_TAG "MyApp"
#define LOGI(...) __android_log_print(ANDROID_LOG_INFO, LOG_TAG, __VA_ARGS__)
#define LOGE(...) __android_log_print(ANDROID_LOG_ERROR, LOG_TAG, __VA_ARGS__)


#define MAX_VERTICES 100000
#define MAX_FACES 100000
#define CANVAS_W 500
#define CANVAS_H 500
#define PALETTE_X 520
#define PALETTE_Y 20
#define PALETTE_SIZE 100 
#define PALETTE_GAP 30


#define CANVAS_SCREEN_X 0
#define CANVAS_SCREEN_Y 0

#define LOOPER_ID_USER 3

#define BUTTON_UP    (1 << 0) //1
#define BUTTON_DOWN  (1 << 1) //2
#define BUTTON_LEFT  (1 << 2) //4
#define BUTTON_RIGHT (1 << 3) //8
#define BUTTON_A     (1 << 4) //16
#define BUTTON_B     (1 << 5) //32

typedef struct
{
    float steering;
    float throttle;

    float gyroX;
    float gyroY;
    float gyroZ;

    uint32_t buttons;
} ControlPacket;

int sock;
struct sockaddr_in server;


float mountainangle;
float mountainoffsetx;
float mountainoffsety;

typedef struct
{
    int x, y;
    int w, h;
    const char *label;
    int pressed;
} Button;

Button btnUp;
Button btnDown;
Button btnLeft;
Button btnRight;
Button btnA;
Button btnB;


int SquareX = -1;
int SquareY = -1;
int touchX = 0;
int touchY = 0;
int touchDown = 0;
uint32_t Canvas[CANVAS_W * CANVAS_H];
void DrawSquare3(int x, int y);
void DrawSquareLine(int x0, int y0, int x1, int y1);
void PutPixel(int x, int y, uint32_t color);
void DrawTouchDot(int x, int y);
uint32_t BrushColor = 0xffffffff;   // brush color variable
int MouseDown = 0;
int LastX = 0;
int LastY = 0;
int activePointer = 0;
int pointerLocked = 0;
int activeColor;


#define MAX_TOUCHES 16

typedef struct
{
    int active;
    int x;
    int y;
} Touch;

Touch touches[MAX_TOUCHES];


ASensorManager * sm;
const ASensor * as;
bool no_sensor_for_gyro = false;
ASensorEventQueue* aeq;
ALooper * l;


short screenx = 0;
short screeny = 0;
float accx, accy, accz;
int accs;
char imuStatus[128] = "IMU: Not initialized";



void UDPInit(const char *ip, int port) 
{
    sock = socket(AF_INET, SOCK_DGRAM, 0);

    int saved_errno = errno;

    LOGI("socket() returned %d errno=%d", sock, saved_errno);

    if(sock < 0)
    {
        LOGE("SOCKET CREATION FAILED");
        return;
    }

    memset(&server, 0, sizeof(server));

    server.sin_family = AF_INET;
    server.sin_port = htons(port);

    int result = inet_pton(AF_INET, ip, &server.sin_addr);

    LOGI("inet_pton result=%d", result);

    if(result <= 0)
    {
        LOGE("Invalid IP address");
    }

    LOGI("UDP ready sock=%d", sock);
}

int UDPSend(float steering, float throttle, uint32_t buttons) 
{
    ControlPacket packet;
    
    packet.steering = steering;
    packet.throttle = throttle;
    
    packet.gyroX = accx;
    packet.gyroY = accy;
    packet.gyroZ = accz;

    packet.buttons = buttons;
    
    
    
    if(sock < 0)
    {
        LOGE("Invalid socket: %d", sock);
        return -1;
    }

    int result = sendto(sock,
	            &packet,
	            sizeof(packet),
	            0,
	            (struct sockaddr*)&server,
	            sizeof(server));

    if(result < 0)
    {
        LOGE("sendto failed sock=%d errno=%d", sock, errno);
    }
    
    
    //int result = sendto(sock, &packet, sizeof(packet), 0, (struct sockaddr*)&server, sizeof(server));
    
    LOGI("UDP SEND result=%d steering=%f throttle=%f buttons=%u",
         result,
         steering,
         throttle,
         buttons);
         
    return result;
    
}

void UDPCleanup()
{
    close(sock);
}

void InitButtons(int screenW, int screenH)
{
    int size = 150;
    int margin = 30;
    int padX = 340;
    int padY = 600;
    int gap = 20;
    
    btnUp = (Button){
        padX,
        padY - size - gap,
        size,
        size,
        "UP",
        0
    };
    
    btnDown = (Button){
        padX,
        padY + size + gap,
        size,
        size,
        "DOWN",
        0
    };
    
    btnLeft = (Button){
        padX - size - gap,
        padY,
        size,
        size,
        "LEFT",
        0
    };
    
    btnRight = (Button){
        padX + size + gap,
        padY,
        size,
        size,
        "RIGHT",
        0
    };

    // A button (lower right)
    btnA = (Button){
        screenW - size - 50,
        screenH - size - 80,
        size,
        size,
        "A",
        0
    };

    // B button offset diagonally
    btnB = (Button){
        screenW - size * 2 - 80,
        screenH - size - 150,
        size,
        size,
        "B",
        0
    };
    
    //LOGI("UP BUTTON x=%d y=%d w=%d h=%d",
         //btnUp.x, btnUp.y, btnUp.w, btnUp.h);
}



void DrawButton(Button *b)
{
    if (b->pressed)
        CNFGColor(0xAAAAAAFF);   // darker when pressed
    else
        CNFGColor(0xFFFFFFFF);

    CNFGTackRectangle(
        b->x,
        b->y,
        b->x + b->w,
        b->y + b->h);

    CNFGColor(0x000000FF);

    CNFGPenX = b->x + 20;
    CNFGPenY = b->y + b->h / 2 + 8;
    CNFGDrawText(b->label, 3);
}

int PointInButton(Button *b, int x, int y)
{
    return (x >= b->x &&
            x <  b->x + b->w &&
            y >= b->y &&
            y <  b->y + b->h);
}

void UpdateButtons()
{
    btnUp.pressed = 0;
    btnDown.pressed = 0;
    btnLeft.pressed = 0;
    btnRight.pressed = 0;
    btnA.pressed = 0;
    btnB.pressed = 0;

    for(int i = 0; i < MAX_TOUCHES; i++)
    {
        if(!touches[i].active)
            continue;

        int x = touches[i].x;
        int y = touches[i].y;

        if(PointInButton(&btnUp, x, y))
            btnUp.pressed = 1;

        if(PointInButton(&btnDown, x, y))
            btnDown.pressed = 1;

        if(PointInButton(&btnLeft, x, y))
            btnLeft.pressed = 1;

        if(PointInButton(&btnRight, x, y))
            btnRight.pressed = 1;

        if(PointInButton(&btnA, x, y))
            btnA.pressed = 1;

        if(PointInButton(&btnB, x, y))
            btnB.pressed = 1;
    }
}
/*
void SetupIMU()
{
	sm = ASensorManager_getInstance();
	as = ASensorManager_getDefaultSensor( sm, ASENSOR_TYPE_GYROSCOPE );
	no_sensor_for_gyro = as == NULL;
	l = ALooper_prepare( ALOOPER_PREPARE_ALLOW_NON_CALLBACKS );
	aeq = ASensorManager_createEventQueue( sm, (ALooper*)&l, 0, 0, 0 ); //XXX??!?! This looks wrong.
	if(!no_sensor_for_gyro) {
		ASensorEventQueue_enableSensor( aeq, as);
		printf( "setEvent Rate: %d\n", ASensorEventQueue_setEventRate( aeq, as, 10000 ) );
	}

}
*/
void SetupIMU()
{
    sm = ASensorManager_getInstance();

    as = ASensorManager_getDefaultSensor(
            sm,
            ASENSOR_TYPE_ACCELEROMETER);

    no_sensor_for_gyro = (as == NULL);

    if(no_sensor_for_gyro)
    {
        sprintf(imuStatus, "No accelerometer");
        return;
    }

    l = ALooper_prepare(ALOOPER_PREPARE_ALLOW_NON_CALLBACKS);

    aeq = ASensorManager_createEventQueue(
            sm,
            (ALooper*)&l,
            0,
            0,
            0);

    if(!aeq)
    {
        sprintf(imuStatus, "Queue failed");
        no_sensor_for_gyro = true;
        return;
    }

    int rc = ASensorEventQueue_enableSensor(aeq, as);

    LOGI("enableSensor rc=%d", rc);

    rc = ASensorEventQueue_setEventRate(
            aeq,
            as,
            10000);

    LOGI("setEventRate rc=%d", rc);

    sprintf(imuStatus, "Accelerometer OK");
}



void AccCheck()
{
    if(no_sensor_for_gyro)
        return;

    ASensorEvent evt;

    do
    {
        int s = ASensorEventQueue_getEvents(
                    aeq,
                    &evt,
                    1);

        if(s <= 0)
            break;

        accx = evt.acceleration.x;
        accy = evt.acceleration.y;
        accz = evt.acceleration.z;

        accs++;

        sprintf(imuStatus,
                "x=%.2f y=%.2f z=%.2f",
                accx,
                accy,
                accz);

    } while(1);
}


unsigned frames = 0;
unsigned long iframeno = 0;

void AndroidDisplayKeyboard(int pShow);

int lastbuttonx = 0;
int lastbuttony = 0;
int lastmotionx = 0;
int lastmotiony = 0;
int lastbid = 0;
int lastmask = 0;
int lastkey, lastkeydown;

static int keyboard_up;

void HandleKey( int keycode, int bDown )
{
	lastkey = keycode;
	lastkeydown = bDown;
	if( keycode == 10 && !bDown ) { keyboard_up = 0; AndroidDisplayKeyboard( keyboard_up );  }

	if( keycode == 4 ) { AndroidSendToBack( 1 ); } //Handle Physical Back Button.
	
        if(!bDown)
        return;
}

void HandleButton(int x, int y, int id, int bDown)
{
    //LOGI("BUTTON: %d %d %d", x, y, bDown);
    if(id >= 0 && id < MAX_TOUCHES) {
        touches[id].active = bDown;
        touches[id].x = x;
        touches[id].y = y;
    }

    lastbuttonx = x;
    lastbuttony = y;
    touchX = x;
    touchY = y;
    touchDown = bDown;
    
    if(!bDown) return;

    int sz = PALETTE_SIZE;
    int gap = PALETTE_GAP;

    if(PointInRect(x,y,PALETTE_X,PALETTE_Y + 0*(sz+gap), sz, sz)) {
        BrushColor = 0xFFFFFFFF; //white
        activeColor = 0;
        }
    else if(PointInRect(x,y,PALETTE_X,PALETTE_Y + 1*(sz+gap), sz, sz)) {
        BrushColor = 0xFF0000FF; //red
        activeColor = 1;
        }
    else if(PointInRect(x,y,PALETTE_X,PALETTE_Y + 2*(sz+gap), sz, sz)) {
        BrushColor = 0x00FF00FF; //green
        activeColor = 2;
        }
    else if(PointInRect(x,y,PALETTE_X,PALETTE_Y + 3*(sz+gap), sz, sz)) {
        BrushColor = 0x0000FFFF; //blue
        activeColor = 3;
        }
}

void HandleMotion(int x, int y, int id)
{
    if(id >= 0 && id < MAX_TOUCHES)
    {
        touches[id].x = x;
        touches[id].y = y;
    }
}

void ClearCanvas(uint32_t color)
{
    for(int i = 0; i < CANVAS_W * CANVAS_H; i++)
        Canvas[i] = color;
}

void DrawPalette()
{
    int x = PALETTE_X;
    int y = PALETTE_Y;

    int sz = PALETTE_SIZE;
    int gap = PALETTE_GAP;

    // white
    CNFGColor(0xffffffff);
    CNFGTackRectangle(x, y + 0*(sz + gap),
                         x + sz, y + 0*(sz + gap) + sz);

    // red
    CNFGColor(0xff0000ff);
    CNFGTackRectangle(x, y + 1*(sz + gap),
                         x + sz, y + 1*(sz + gap) + sz);

    // green
    CNFGColor(0x00ff00ff);
    CNFGTackRectangle(x, y + 2*(sz + gap),
                         x + sz, y + 2*(sz + gap) + sz);

    // blue
    CNFGColor(0x0000ffff);
    CNFGTackRectangle(x, y + 3*(sz + gap),
                         x + sz, y + 3*(sz + gap) + sz);
}
/*
void DrawPalette()
{
    int x = PALETTE_X;
    int y = PALETTE_Y;

    CNFGColor(0xffffffff);
    CNFGTackRectangle(x, y, x+40, y+40); // white

    CNFGColor(0xff0000ff);
    CNFGTackRectangle(x, y+50, x+40, y+90); // red

    CNFGColor(0x00ff00ff);
    CNFGTackRectangle(x, y+100, x+40, y+140); // green

    CNFGColor(0x0000ffff);
    CNFGTackRectangle(x, y+150, x+40, y+190); // blue
}*/

int PointInRect(int x, int y, int rx, int ry, int rw, int rh)
{
    return (x >= rx && x < rx + rw && y >= ry && y < ry + rh);
}


void DrawTouchDot(int x, int y)
{
    int cx = ScreenToCanvasX(x);
    int cy = ScreenToCanvasY(y);

    for(int yy = -5; yy <= 5; yy++)
    {
        for(int xx = -5; xx <= 5; xx++)
        {
            PutPixel(cx + xx, cy + yy, BrushColor); 
        }
    }
}

void PutPixel(int x, int y, uint32_t color)
{
    if(x < 0 || x >= CANVAS_W)
        return;

    if(y < 0 || y >= CANVAS_H)
        return;

    Canvas[y * CANVAS_W + x] = color;
}


void DrawSquare3(int x, int y)
{
    for(int yy = 0; yy < 20; yy++)
    {
        for(int xx = 0; xx < 20; xx++)
        {
            PutPixel(x + xx, y + yy, BrushColor);
        }
    }
}


extern struct android_app * gapp;

void HandleDestroy()
{
	printf( "Destroying\n" );
	exit(10);
}

volatile int suspended;

void HandleSuspend()
{
	suspended = 1;
}

void HandleResume()
{
	suspended = 0;
}

void DrawSquare2(void)
{
    if(SquareX < 0)
        return;

    //CNFGColor(0xffffffff);   // green

    CNFGTackRectangle(
        SquareX - 10,
        SquareY - 10,
        SquareX + 10,
        SquareY + 10);
}

int ScreenToCanvasX(int x)
{
    return x - CANVAS_SCREEN_X;
}

int ScreenToCanvasY(int y)
{
    return y - CANVAS_SCREEN_Y;
}

void DrawSquareLine(int x0, int y0, int x1, int y1)
{
    int dx = abs(x1 - x0);
    int sx = x0 < x1 ? 1 : -1;

    int dy = -abs(y1 - y0);
    int sy = y0 < y1 ? 1 : -1;

    int err = dx + dy;

    while(1)
    {
        DrawSquare3(x0, y0);  // stamp square

        if(x0 == x1 && y0 == y1)
            break;

        int e2 = err * 2;

        if(e2 >= dy)
        {
            err += dy;
            x0 += sx;
        }

        if(e2 <= dx)
        {
            err += dx;
            y0 += sy;
        }
    }
}


int main()
{
	int x, y;
	double ThisTime;
	double LastFPSTime = OGGetAbsoluteTime();
	int linesegs = 0;

	CNFGBGColor = 0x000010ff;
	CNFGSetupFullscreen( "RawDrawAndroid Paint", 1 );

	ClearCanvas(0x000000ff);    // clear to black canvas
	
	CNFGGetDimensions( &screenx, &screeny );
	InitButtons(screenx, screeny);
	//CNFGTackRectangle(0, 0, 100, 100);
	//glEnable(GL_DEPTH_TEST); // enable Z-buffer

	UDPInit("192.168.1.68", 5005);
    	//program = createProgram(vertexShaderSrc, fragmentShaderSrc);
    	//SetupBuffers();

	
	//LoadOBJ("bunny.obj");
	//LOGI("vertexCount: %d", vertexCount);
	//LOGI("indexCount: %d", indexCount);
	float modelScale = 0.0f;
	float angle = 0.0f;
	//FitModelToCamera(vertices, vertexCount, &camera, &modelScale);
	//FitModelToScreen(vertices, vertexCount, &camera, NULL);
	//UploadModel();
	
	
	
/*
	const char * assettext = "Not Found";
	
	AAsset * file = AAssetManager_open( gapp->activity->assetManager, "asset.txt", AASSET_MODE_BUFFER );
	if( file )
	{
		size_t fileLength = AAsset_getLength(file);
		char * temp = malloc( fileLength + 1);
		memcpy( temp, AAsset_getBuffer( file ), fileLength );
		temp[fileLength] = 0;
		assettext = temp;
	}
	*/
	SetupIMU();


	while(1)
	{
		int i, pos;
		iframeno++;

		CNFGHandleInput();
		AccCheck();

		if( suspended ) { usleep(50000); continue; }

		

		//InitFaceButtons(screenx, screeny);
		
		CNFGClearFrame();
		
		if(touchDown)
		{
		    int x = touchX;
		    int y = touchY;
		    
		    DrawTouchDot(x, y);
		}
		
		//DrawSquare2();
		//DrawPalette();
		//CNFGBlitImage(Canvas, 0, 0, CANVAS_W, CANVAS_H);
		UpdateButtons();
		
		//DrawButton(&btnUp);
		DrawButton(&btnA);
		DrawButton(&btnB);
		DrawButton(&btnUp);
		DrawButton(&btnDown);
		DrawButton(&btnLeft);
		DrawButton(&btnRight);
		//DrawButton(&btnDown);
		//DrawButton(&btnLeft);
		//DrawButton(&btnRight);
		
		float throttle = 0.0f;
		float steering = 0.0f;
		
		if (btnUp.pressed)
		{
		    throttle = 1.0f;
		}
		uint32_t buttons = 0;

		if(btnUp.pressed)
		    buttons |= BUTTON_UP;

		if(btnDown.pressed)
		    buttons |= BUTTON_DOWN;

		if(btnLeft.pressed)
		    buttons |= BUTTON_LEFT;

		if(btnRight.pressed)
		    buttons |= BUTTON_RIGHT;

		if(btnA.pressed)
		    buttons |= BUTTON_A;

		if(btnB.pressed)
		    buttons |= BUTTON_B;

		int sendResult = UDPSend(steering, throttle, buttons);
		
		CNFGColor(0xffffffff);
		CNFGPenX = 0; CNFGPenY = 600;
		
		CNFGColor(0xffffffff);
		CNFGPenX = 10;
		CNFGPenY = 150;
		CNFGDrawText(imuStatus, 3);
		
		char debug[128];

		sprintf(debug, "Touch %d %d Down %d Up %d Send %d",
			touchX,
			touchY,
			touchDown,
			btnUp.pressed,
			sendResult);

		CNFGColor(0xffffffff);
		CNFGPenX = 10;
		CNFGPenY = 50;
		CNFGDrawText(debug, 3);
		
		char debug2[128];

		sprintf(debug2,
			"Send=%d errno=%d",
			sendResult,
			errno);
		CNFGPenX = 10;
		CNFGPenY = 80;
		CNFGDrawText(debug2, 3);
		
		float scale = 300.0f;

		int centerX = screenx / 2;
		int centerY = screeny / 2;

		int dotX = centerX + (int)(accx * scale);
		int dotY = centerY + (int)(accy * scale);
		
		CNFGColor(0x00FF00FF);   // green

		CNFGTackRectangle(
		    dotX - 10,
		    dotY - 10,
		    dotX + 10,
		    dotY + 10);
		    
		CNFGColor(0xFFFFFFFF);

		CNFGTackSegment(centerX - 20, centerY,
				centerX + 20, centerY);

		CNFGTackSegment(centerX, centerY - 20,
				centerX, centerY + 20);
		
		char imuDebug[128];

		sprintf(imuDebug,
		    "gyro x=%.3f y=%.3f z=%.3f accs=%d",
		    accx,
		    accy,
		    accz,
		    accs);

		CNFGPenX = 10;
		CNFGPenY = 120;
		CNFGDrawText(imuDebug, 3);
		
		//char st[50];
		//sprintf( st, "%dx%d %d %d %d %d %d %d\n%d %d\n%5.2f %5.2f %5.2f %d\n %d\n %08X", screenx, screeny, lastbuttonx, lastbuttony, lastmotionx, lastmotiony, lastkey, lastkeydown, lastbid, lastmask, accx, accy, accz, accs, activeColor, BrushColor );
		//CNFGDrawText( st, 10 );
		//CNFGSetLineWidth( 2 );

		//glBindFramebuffer(GL_FRAMEBUFFER, 0);


		//glViewport(0, 0, screenx, screeny);
		
		//glClearColor(0.1f,0.1f,0.3f,1.0f);
		//glClear(GL_COLOR_BUFFER_BIT);
		//glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT);
		
		//RenderFrame(angle);
		//angle += 0.01f; // slow rotation
		
		//On Android, CNFGSwapBuffers must be called, and CNFGUpdateScreenWithBitmap does not have an implied framebuffer swap.
		CNFGSwapBuffers(); 
		
		frames++;
		
		ThisTime = OGGetAbsoluteTime();
		if( ThisTime > LastFPSTime + 1 )
		{
			//printf( "FPS: %d\n", frames );
			frames = 0;
			linesegs = 0;
			LastFPSTime+=1;
		}
		
	}
	
	UDPCleanup();

	return(0);
}
