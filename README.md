# Aplicaciones-de-Procesos-e-Hilos
Aplicacion de un juego basico utilizando hilos y X11 en C

# 1. Aplicacion grafica X11

Crear un directorio para el proyecto
```bash
mkdir graficos
```
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

....
//Llamando a X11
Display *display = XOpenDisplay(NULL);
    if (!display) {
        fprintf(stderr, "Error al conectar con X11.\n");
        exit(1);
    }
    
    int pantalla = DefaultScreen(display);
    Window raiz = RootWindow(display, pantalla);
    Window ventana = XCreateSimpleWindow(display, raiz, 10, 10, ANCHO, ALTO, 1, 0, 0);
    
    XStoreName(display, ventana, "Caza de enemigos con el mouse en X11");
    
    XSelectInput(display, ventana, ExposureMask | KeyPressMask | PointerMotionMask 
                 | ButtonPressMask);
    
    XMapWindow(display, ventana);

    XImage *x_image = XCreateImage(display, DefaultVisual(display, pantalla), 
                                   DefaultDepth(display, pantalla), ZPixmap, 0, 
                                   (char*)buffer_pixeles, ANCHO, ALTO, 32, 0);
    GC gc = DefaultGC(display, pantalla);
```

Compilar y linkear el archivo
```bash
gcc main.c -lX11 -lpthread -o ventana
```

Compilar y linkear
gcc main.c -lX11 -lpthread -o juego_simple


