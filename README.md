# Aplicaciones-de-Procesos-e-Hilos
Aplicaciones en C para procesos e hilos

# 1. Aplicacion grafica X11

Crear un archivo con extension c (eg. main.c)

```bash
touch main.c
```
Para este tutorial se utilizara el protocolo X11 para comunicar las aplicaciones con la pantalla
```c
#include <X11/Xlib.h>
#include <X11/Xutil.h>
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <unistd.h>

#define ANCHO 640
#define ALTO 480

// Estructura para representar nuestro búfer de píxeles en RAM (Color: 0x00RRGGBB)
uint32_t buffer_pixeles[ANCHO * ALTO];

int main() {
    // 1. Conectarse al servidor gráfico X11 (WSL2 se conecta automáticamente aquí)
    Display *display = XOpenDisplay(NULL);
    if (display == NULL) {
        fprintf(stderr, "Error: No se pudo abrir el display de X11.\n");
        //Asegúrate de que WSLg está activo.
        return 1;
    }

    int pantalla = DefaultScreen(display);
    Window raiz = RootWindow(display, pantalla);

    // 2. Crear la ventana nativa
    Window ventana = XCreateSimpleWindow(display, raiz, 10, 10, ANCHO, ALTO, 1, 
                                        BlackPixel(display, pantalla), 
                                        BlackPixel(display, pantalla));

    XStoreName(display, ventana, "Ventana Nativa X11 (Píxeles y Sprites)");

    // 3. Seleccionar los eventos que queremos escuchar (Teclado y 
    // Exposición/Redibujado)
    XSelectInput(display, ventana, ExposureMask | KeyPressMask);
    XMapWindow(display, ventana);

    // 4. Crear una imagen XImage para poder volcar nuestro búfer de 
    // píxeles directo a la ventana
    Visual *visual = DefaultVisual(display, pantalla);
    int prof = DefaultDepth(display, pantalla);
    
    XImage *x_image = XCreateImage(display, visual, prof, ZPixmap, 0, 
                                   (char*)buffer_pixeles, ANCHO, ALTO, 32, 0);

    GC gc = DefaultGC(display, pantalla);
    int ejecutando = 1;

    // 5. Bucle principal del juego/aplicación
    while (ejecutando) {

        // LÓGICA DE RENDERIZADO POR SOFTWARE:
        // Limpiar la pantalla (pintar todo de azul oscuro como fondo)
        for (int i = 0; i < ANCHO * ALTO; i++) {
            buffer_pixeles[i] = 0x001A1A3A; // Formato Hexadecimal: 0x00RRGGBB
        }

        // Volcar nuestro búfer de píxeles de la RAM hacia la ventana de X11
        XPutImage(display, ventana, gc, x_image, 0, 0, 0, 0, ANCHO, ALTO);
        XFlush(display);

        // Controlar la velocidad del bucle (aproximadamente 60 FPS)
        usleep(16666); 
    }

    // 6. Limpieza de recursos
    XDestroyImage(x_image); // Esto también libera buffer_pixeles internamente
    XDestroyWindow(display, ventana);
    XCloseDisplay(display);

    return 0;
}
```

Dibujar un sprite, darle un efecto al fondo de la ventana y agregar eventos...

# 2. Aplicacion de un juego de punteria utilizando dos hilos (fisica y render)

```c
#include <X11/Xlib.h>
#include <X11/Xutil.h>
#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <unistd.h>
#include <pthread.h>
#include <time.h>

#define ANCHO 800
#define ALTO 600
#define TAMANO_ENEMIGO 40
#define CANTIDAD_ENEMIGOS 8

typedef struct {
    float x, y;
    float vel_x, vel_y;
    int estado;          // 0 = Volando, 1 = Destruido   
} Enemigo;

// Variables Globales
uint32_t buffer_pixeles[ANCHO * ALTO];
Enemigo lista_enemigos[CANTIDAD_ENEMIGOS];
int mouse_x = 0;
int mouse_y = 0;
int clic_detectado = 0;
int ejecutando = 1;

pthread_mutex_t mutex_datos;

void pintar_pixel(int x, int y, uint32_t color) {
    if (x >= 0 && x < ANCHO && y >= 0 && y < ALTO) {
        buffer_pixeles[y * ANCHO + x] = color;
    }
}

// Dibuja una silueta cuadrada con un patrón interno
void dibujar_silueta_enemigo(int px, int py, int t, uint32_t color) {
    for (int y = 0; y < t; y++) {
        for (int x = 0; x < t; x++) {
            pintar_pixel(px + x, py + y, color);
        }
    }
}

// ==========================================
// HILO DE FÍSICA E IMPACTOS (Para todos los enemigos)
// ==========================================
void* hilo_fisica(void* arg) {
    while (ejecutando) {
        pthread_mutex_lock(&mutex_datos);

        // Procesar clic del mouse (Hit Detection)
        if (clic_detectado) {
            for (int i = 0; i < CANTIDAD_ENEMIGOS; i++) {
                if (lista_enemigos[i].estado == 0) { // Solo si está vivo
                    if (mouse_x >= lista_enemigos[i].x && mouse_x <= (lista_enemigos[i].x + TAMANO_ENEMIGO) &&
                        mouse_y >= lista_enemigos[i].y && mouse_y <= (lista_enemigos[i].y + TAMANO_ENEMIGO)) {
                        
                        lista_enemigos[i].estado = 1; // Cambiar a estado explosión
                        break; // Un clic solo destruye un enemigo a la vez
                    }
                }
            }
            clic_detectado = 0; // Consumir clic
        }

        // Actualizar movimientos
        for (int i = 0; i < CANTIDAD_ENEMIGOS; i++) {
            if (lista_enemigos[i].estado == 0) {
                // Mover enemigo
                lista_enemigos[i].x += lista_enemigos[i].vel_x;
                lista_enemigos[i].y += lista_enemigos[i].vel_y;

                // Rebotar contra los bordes
                if (lista_enemigos[i].x <= 0 || lista_enemigos[i].x + TAMANO_ENEMIGO >= ANCHO) {
                    lista_enemigos[i].vel_x *= -1;
                }
                if (lista_enemigos[i].y <= 0 || lista_enemigos[i].y + TAMANO_ENEMIGO >= ALTO) {
                    lista_enemigos[i].vel_y *= -1;
                }
            }
        }

        pthread_mutex_unlock(&mutex_datos);
        usleep(10000); // 100 Hz
    }
    return NULL;
}

// ==========================================
// HILO DE RENDERIZADO (Dibujar la escena)
// ==========================================
void* hilo_render(void* arg) {
    Display *display = XOpenDisplay(NULL);
    if (!display) {
        fprintf(stderr, "Error al conectar con X11.\n");
        exit(1);
    }
    
    int pantalla = DefaultScreen(display);
    Window raiz = RootWindow(display, pantalla);
    Window ventana = XCreateSimpleWindow(display, raiz, 10, 10, ANCHO, ALTO, 1, 0, 0);
    
    XStoreName(display, ventana, "Caza de Siluetas Mutli-Enemigo en (X11)");
    XSelectInput(display, ventana, ExposureMask | KeyPressMask | PointerMotionMask | ButtonPressMask);
    XMapWindow(display, ventana);

    XImage *x_image = XCreateImage(display, DefaultVisual(display, pantalla), 
                                   DefaultDepth(display, pantalla), ZPixmap, 0, 
                                   (char*)buffer_pixeles, ANCHO, ALTO, 32, 0);
    GC gc = DefaultGC(display, pantalla);
    XEvent evento;

    while (ejecutando) {
        while (XPending(display) > 0) {
            XNextEvent(display, &evento);
            pthread_mutex_lock(&mutex_datos);
            if (evento.type == MotionNotify) {
                mouse_x = evento.xmotion.x;
                mouse_y = evento.xmotion.y;
            }
            if (evento.type == ButtonPress && evento.xbutton.button == Button1) {
                clic_detectado = 1;
            }
            if (evento.type == KeyPress) {
                if (XLookupKeysym(&evento.xkey, 0) == XK_Escape) ejecutando = 0;
            }
            pthread_mutex_unlock(&mutex_datos);
        }

        // Fondo Gris Oscuro Industrial
        for (int i = 0; i < ANCHO * ALTO; i++) buffer_pixeles[i] = 0x00151515;

        pthread_mutex_lock(&mutex_datos);
        
        // Dibujar a todo el grupo de enemigos
        for (int i = 0; i < CANTIDAD_ENEMIGOS; i++) {
            if (lista_enemigos[i].estado == 0) {
                // Siluetas Cyan para enemigos activos
                dibujar_silueta_enemigo((int)lista_enemigos[i].x, (int)lista_enemigos[i].y, TAMANO_ENEMIGO, 0x0000FFFF);
            }
        }

        pthread_mutex_unlock(&mutex_datos);

        XPutImage(display, ventana, gc, x_image, 0, 0, 0, 0, ANCHO, ALTO);
        XFlush(display);

        usleep(16666); // ~60 FPS
    }

    XDestroyImage(x_image);
    XDestroyWindow(display, ventana);
    XCloseDisplay(display);
    return NULL;
}

// ==========================================
// CONFIGURACIÓN INICIAL
// ==========================================
int main() {
    srand(time(NULL));
    pthread_mutex_init(&mutex_datos, NULL);

    // Inicializar cada enemigo con posiciones y aleatorias
    for (int i = 0; i < CANTIDAD_ENEMIGOS; i++) {
        lista_enemigos[i].x = rand() % (ANCHO - TAMANO_ENEMIGO - 50) + 25;
        lista_enemigos[i].y = rand() % (ALTO - TAMANO_ENEMIGO - 50) + 25;
        
        // Velocidades aleatorias entre 2.0 y 5.0 (positivas o negativas)
        lista_enemigos[i].vel_x = (float)(rand() % 4 + 2) * ((rand() % 2 == 0) ? 1 : -1);
        lista_enemigos[i].vel_y = (float)(rand() % 4 + 2) * ((rand() % 2 == 0) ? 1 : -1);
        
        lista_enemigos[i].estado = 0;
    }

    pthread_t h_fisica, h_render;
    pthread_create(&h_fisica, NULL, hilo_fisica, NULL); // Nota técnica: hilo_fisica en tu código
    pthread_create(&h_render, NULL, hilo_render, NULL);

    pthread_join(h_fisica, NULL);
    pthread_join(h_render, NULL);

    pthread_mutex_destroy(&mutex_datos);
    printf("¡Simulación multienemigo finalizada!\n");
    return 0;
}
```



