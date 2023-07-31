# ZXTRES wrapper (ZX3W)
Este documento intenta ser una guía para saber cómo incluir y conectar el ZXTRES wrapper a un core existente.
ZXTRES wrapper (desde ahora, ZX3W) permite a un core:

 - Poder sacar imagen en VGA 60 Hz y DisplayPort
 - Sacar sonido tanto a través del soporte de sigma-delta de 1 bit, heredado de ZXUNO y ZXDOS, como sonido a través del DAC I2S (y pronto también por DisplayPort, me está costando, pero poquito a poco :) )
 - Interrogar al sistema sobre los parámetros de video almacenados en BIOS
 - Gestionar la interfaz de joystick para los dos puertos de joystick.
 - Permitir, con una señal, resetear la FPGA y volver al core principal de Spectrum.

## Prerrequisitos
La presente versión de ZX3W es universal para los tres modelos de FPGA. El módulo detecta para qué FPGA se está sintetizando merced a una macro que debe definirse en las opciones del sintetizador (luego hablaré sobre esto)
Partimos de un core que originalmente, genera:
 - Salida RGB a 15 kHz en PAL o NTSC.
 - Sonido de 1 a 16 bits, mono o estéreo, a base de generar muestras PCM (las de siempre, de toda la vida). Inicialmente ZX3W supone que las muestras, cuando ocupan más de 1 bit, están en complemento a 2. Luego daré pautas para convertir otro tipo de muestras, si fuera necesario.
 - El core debe tener una señal de rerset, que inhabilite por completo a todo el core entero.
 - El core debe proporcionar al menos un reloj, que llamaremos "reloj de pixel" o "dotclock" que se supone que se genera en el circuito de video del propio core. Este reloj puede ser un reloj tal cual, o puede que sea un reloj más rápido, junto con una señal de enable.
 - Opcionalmente, el core puede generar otro reloj, fijo a 100 MHz, y se usa para la generación de los relojes de color PAL y NTSC para el módulo de video compuesto, externo a ZXTRES.
 - ZX3W no se encarga de gestionar el teclado ni el ratón, al menos en la presente versión. Esto es porque la gran mayoría de cores ya tienen su propia gestión del teclado PS/2, y del ratón los que también lo tengan. Si acaso, para los cores de arcade que, idealmente, no deberían tener nada de esto metido dentro, habilitaré una opción para control básico de teclado en posteriores revisiones.

## Limitaciones
- No tenemos aún sonido vía DisplayPort. Paciencia, ya llegará.
- La resolución está fijada a 640x480. En algunos cores esto no será problemático (ej. el Spectrum, donde el borde es muy grueso) pero en otros se podría perder información en la zona no mostrada. En versiones siguientes podré ofrecer otras resoluciones como 800x600 .

  No quiero irme más allá de esta resolución, porque irse a resoluciones mayores implica, en DisplayPort, conmutar a una frecuencia de 2.70 GHz . Ahora estamos trabajando a 1.62 GHz, lo que significa un reloj de 81 MHz para la gestión del contenido en DisplayPort (1.62 GHz / 20). La frecuencia del reloj de pixel, idealmente, debería ser un divisor entero de estos 81 MHz.

  El modo de 640x480 que ofrece ahora el wrapper usa 27 MHz (81 / 3), que aunque está un poco alejado de los estándares 25 MHz, no parece que haya problemas con los monitores probados hasta ahora. Un modo de 800x600 a 60 Hz es también posible con esta cvonfiguración, ya que usa un reloj de pixel de 40 MHz (40.5 MHz usaríamos en realidad).

  Para poder usar resoluciones mayores, como 1024x768, los 81 MHz del reloj de gestión no son suficientes, y de ahí tener que cambiar a 2.70 GHz. El reloj de gestión en este caso sería de 2.70 / 20 = 135 MHz. Este reloj ya es lo suficientemente rápido como para que comiencen a aparecer problemas de timing closure en partes del circuito con mucha lógica.
- El core debe proveer de una salida con timings PAL o NTSC, y con sincronismos horizontal y vertical separados. Particularmente, el sincronismo vertical debería ser un pulso único, no un tren de pulsos. ZX3W usa estos dos sincronismos para medir el número de píxeles por linea, y el número de líneas por frame, para luego determinar dónde comienza la pantalla y saber por tanto a partir de dónde hay que empezar a grabar en el framebuffer.

## Instalación en el código fuente del core
ZX3W está pensado para ser usado desde Vivado versión 2023.1 o superior. Vamos a partir de que tienes un core, ya funcionando de forma básica en ZXTRES. Esto es, que puede sacar video por RGB a 15 kHz, y puedes ver su salida usando el adaptador VGA-SCART de ZXTRES.
 - Copia por completo la carpeta zx3w (copia la carpeta en sí, no su contenido) dentro de la carpeta donde tengas el cóigo fuente del core.
 - Ya dentro de Vivado, añade todos los ficheros de la carpeta zx3w al core. Al principio, aparecerán fuera del arbol de ficheros de tu core. Hasta que no lo instanciemos, estos ficheros no harán nada.
 - Si no lo has hecho antes, pienso que es buena idea crear tres "Runs". En Vivado, un "Run" es un conjunto de pasos de síntesis e implementación, personalizados a tu antojo para, por ejemplo, una FPGA diferente. De esta forma, el mismo core para cada modelo de ZXTRES se sintetizará sin necesidad de un proyecto por separado, o tocar manualmente cosas en el proyecto. En la imagen puedes ver que tengo un "run" por cada FPGA, consistiendo cada uno en una fase de síntesis y otra de implementación. De los tres "Design Runs", tengo activo el de A200T, por lo que, por defecto, sintetizo/implemento/simulo para esa FPGA. Puedes cambiar cuál es tu "run" activo cuando quieras. Puedes incluso lanzar los 3 a la vez, y dejar que se sinteticen todos los cores.

Para crear un nuevo "run", es más fácil partir de una copia del que ya tengas. En cualquier proyecto Vivado se crea un "run" por defecto, llamado synth_1 e impl_1. synth_1 controla qué se hace durante la síntesis, e impl_1 lo que se hace durante la implementación (place & route, etc). Cuando seleccionas un "synth_loquesea" o un "impl_loquesea", en la ventana General, justo encima de donde aparecen todos los "runs", se muestra el nombre del mismo y puedes cambiarlo. Para cada "run" debes cambiar la FPGA para la que sintetiza e implementa (un cambio en synth y otro en impl).

Tener "runs" separados para cada FPGA nos permite personalizar algunos aspectos de la síntesis. Uno de ellos es poder decirle al código fuente del core, para qué FPGA estamos sintetizando el cóigo en ESE "run". Para ello:

![](https://github.com/zxtres/zxtres_wrapper/blob/main/images/runs.png)

 - En la parte de síntesis de cada "run", dando con el botón derecho del ratón y eligiendo "Change Run Settings"...

![](https://github.com/zxtres/zxtres_wrapper/blob/main/images/change_run_settings.png)

Llegamos a esta ventana, donde al final, en "More Options", añadimos una linea como ésta:
```
-verilog_define A200T
```
![](https://github.com/zxtres/zxtres_wrapper/blob/main/images/run_settings.png)

Cambiando el A200T por A35T o A100T, según para qué FPGA estés sintetizando en ese "run". Esto lo que hace es crear un ```#define``` parecido a los de C, que dentro del core yo puedo usar para añadir/cambiar/quitar condicionalmente algunos módulos según a qué FPGA vayas a sintetizar. A diferencia del C, no he conseguido averiguar si el propio Vivado te da alguna opción, en formna de símbolos predefinidos, de saber los detalles de a qué FPGA se sintetiza, asíque de momento usaremos esto. Si no se define nada, se asume A35T.

## Señales de ZX3W
El módulo TLD del wrapper se llama zxtres_wrapper. Está escrito en Verilog. Esta es su definición (cabecera). A medida que la voy presentando, voy explicando para qué sirve cada cosa:
```
module zxtres_wrapper (
  input wire clkvideo,
  input wire enclkvideo,
  input wire clkpalntsc,
  input wire reset_n,
  input wire reboot_fpga,
  //////////////////////////////////////////
```
  **clkvideo** es el dotclock, o reloj de pixel. En el core, en algún sitio debe haber un módulo que produzca la imagen que se ve en pantalla. Si el core es de un arcade ochentero, o de un microordenador, muy probablemente saque salida apta para TV, es decir, RGB a 15 kHz de frecuencia horizontal. Todo eso debe gobernarse mediante un reloj, que o bien es algún tipo de reloj general junto con una señal de enable, o es un reloj dedicado.
  Tanto si es una cosa  como si otra, en clkvideo ponemos esa señal de reloj.

  **enclkvideo** es la señal de habilitación del reloj anterior. Si existe, se pone aquí. Si no existe, aquí ponemos un 1.

  **clkpalntsc** es el reloj opcional de 100 MHz para que el ZX3W genere las señales de reloj de color PAL y NTSC. Si no generamos estos relojes, ponemos aquí un 0.

  **reset_n** es una señal de reset a nivel bajo para todo el wrapper. Normalmente esta señal se conectará a la señal "locked" del MMCM. La señal locked la produce el MMCM cuando la señal de reloj que genera ya es estable. Si no la usamos, ponemos aquí un 1.

  **reboot_fpga** es una señal que si está a 1, hace que la FPGA se resetee y cargue de nuevo el core principal. Lai dea es que una vez que se ponga a 1, ya se quede con ese valor, hasta que la propia FPGA se resetee. No vale con ponerla a 1 durante un ciclo de reloj y luego volverla a 0. Si no la usamos, poner un 0 aquí.
```
  input wire [20:0] sram_addr_in,
  output wire [20:0] sram_addr_out,
  input wire sram_we_n_in,
  output wire sram_we_n_out,
  input wire [7:0] sram_data,
  output wire poweron_reset,
  //////////////////////////////////////////
```
  Este grupo de señales es una interfaz simple con la memoria RAM. Sirven para que el wrapper pueda interrogar, al principio del todo, cuáles son los parámetros de video de la BIOS, y copiarlos para usarlos en el propio wrapper.
  Cuando desde la BIOS, o desde el comando .core de ESXDOS, se pide arrancar un nuevo core, se guarda en la dirección de memoria SRAM 0xZZZZ el valor del [registro $0B del ZXUNO (**SCANDBLCTRL**)](https://www.zxuno.com/wiki/index.php/ZX_Spectrum#Nuevos_registros_E.2FS_para_control_de_ZX-Uno)

  **sram_addr_in**  es el bus de direcciones de la SRAM. Si nuestro core usa la SRAM, conectamos el bus de direcciones ahí. Si el core no usa SRAM, ponemos ahí todo unos. Algo como: 21'h1FFFFF.

  **sram_addr_out** es la salida del bus de direcciones desde el core hasta la SRAM física. Aquí esto lo conectamos directamente al bus de direcciones en el TLD del módulo. Incluso si el core no usa la SRAM para nada, el sistema de interrogación de parámetros de la BIOS sí que lo usa.

  **sram_we_n_in** y **sram_we_n_out** hacen el mismo papel que en el caso anterior, pero para la señal de habilitación de escritura, que es activa a nivel bajo. Si nuestro core no usa la SRAM, poner en sram_we_n_in un 1. sram_we_n_out va conectada directamente afuera de la FPGA, a la SRAM.

**sram_data** es el bus de direcciones de 8 bits, pero el de entrada. Este se conecta directamente al bus de datos bidireccional en el TLD del módulo, conectado directamente a la SRAM. Si el core usa la SRAM internamente, no habrá ningún problema en que ambos, el core y el zx3w tengan señales que conecten a la vez al bus de datos bidireccional.

**poweron_reset** es una señal de salida, activa a nivel alto. Sirve para resetear el core completo (no el ZX3W, sino el core original), con objeto de que dé tiempo al wrapper a interrogar a la memoria para conocer la configuración de video, y por otra parte para dar tiempo a que el reloj de alta velocidad de 135 MHz (y de ahí, el de 1.6 2 GHz) del DisplayPort tengan tiempo de estabilizarse. En el core original, mientras esté en reset, no debería hacerse ninguna operación de memoria, dejando buses de salida de datos a memoria en alta impedancia y señales de control a nivel alto.
```
  output wire config_vga_on,
  output wire config_scanlines_on,
  input wire video_output_sel,
  input wire disable_scanlines,
  input wire [1:0] monochrome_sel,
  input wire interlaced_image,
  input wire ad724_modo,
  input wire ad724_clken,
  //////////////////////////////////////////
```

  **config_vga_on** y **config_scanlines_on** son señales de salida que indican lo que hay guardado en la configuración de la BIOS respecto de si se habilita la salida VGA y/o si dicha salida tiene las scanlines activadas.

Idealmente, conectaremos **config_vga_on** a **video_output_sel** y **config_scanlines_on** a **disable_scanlines** negando esta última señal previamente.

**monochrome_sel** puede tomar los valores 0, 1, 2, 3 y sirve para aplicar cuatro efectos diferentes a la salida de video, que se verán reflejados tanto en la salida original de 15 kHz como en VGA y DP.

0: señal original, sin efectos.
1: señal imitando un monitor de fóforo verde.
2: señal imitando un monitor en fóforo ámbar
3: señal imitando un monitor en blanco y negro.

**interlaced_image** indica, si vale 1, que la imagen de entrada tiene campo para e impar diferentes. Esto sólo lo usará el A200T, que es quien tiene memoria suficiente para un framebuffer completo, pero no tiene por qué ser siempre así (luego elaboro esto).

**ad724_modo** vale 0 para indicar al generador de reloj de color que se ha de generar un reloj PAL (17.73 MHz) si vale 0, o NTSC (14.318 MHz) si vale 1. Si no se usa la generación de relojes de color, poner aquí un 0.

**ad724_clken** indica a ZX3W que el reloj de color base es correcto y que puede proceder a generar los relojes de color. Si anteriormente decidimos que clkntscpal iba a ser 0, aquí también pondremos 0.

```
  input wire [7:0] ri,
  input wire [7:0] gi,
  input wire [7:0] bi,
  input wire hsync_ext_n,
  input wire vsync_ext_n,
  input wire csync_ext_n,
  //////////////////////////////////////////
```
**atención a estas señales**: ri, gi, bi son los valores RGB de entrada. Trabajamos con 8 bits por color primario. Si el core no produce esa cantidad de bits por color primario, lo que se hace es escalar los valores de la siguiente forma:

Tenemos un valor de N bits y queremos convertirlo a su equivalente en M bits, con M>N (en este caso, M = 8). Pues cogemos el valor original de N bits, y lo concatenamos consigo mismo, tantas veces como sea necesario hasta llegar a los M bits. Si en la última concatenación, nos pasamos de bits, pues cortamos para llegar a exactamente M bits.

Ejemplo: tenemos la señal ```wire [2:0] red``` de 3 bits, y queremos pasarla a ```ri``` que es de 8 bits. Pues haremos: ```{red, red, red[2:1]}```

Es posible que el core que estás adaptando tenga una paleta fija de colores, por ejemplo, 16 colores diferentes, como el Commodore 64. Y es una paleta fija, no la puede cambiar el usuario. En ese caso, se puede conseguir una reducción espectacular de memoria del framebuffer (permitiendo así un framebuffer completo para A100T y A35T) enviando el cóigo del color (4 bits) y siendo eso lo que se grabe en el framebuffer.

**hsync_ext_n** es el sincronismo horizontal original del core. En algunos cores está presente de forma natural, y en otros hay que sacarlo de la circuitería de video, porque no se genera de forma independiente, sino ya mezclado con el vertical.
**vsync_ext_n** es el sincronismo vertical original  del core. Mismas consideraciones que en el caso anterior. Debe ser un pulso único, no un tren de pulsos con pre y post ecualización, ni nada de eso.
**csync_ext_n** es el sincronismo compuesto original del core. Mismas consideraciones que en el caso anterior.
```
  input wire [15:0] audio_l,
  input wire [15:0] audio_r,
  //////////////////////////////////////////
```
Por aquí entran las muestras de audio. ZX3W necesita en principio muestras de 16 bits, en complemento a 2. Si tu core sólo genera sonido de 1 bit (como el beep del Spectrum) puedes generar estas dos señales así (*beep* es aquí una señal de 1 bit que contiene el audio mono original):
```
reg [15:0] left, right;
always @* begin
  left = (beep == 1'b0)? 16'h8000 : 16'h7FFF;
  right = left;
end
```
Si genera una señal monoaural, proveniente de un chip de sonido que genere salidas de, por ejemplo, 4 bits en binario natural (no complemento a 2), entonces puede convertirse a 16 bits y complemento a 2, así (*chipsonido* es aquí una señal de 4 bits que contiene el audio mono original):
```
reg [15:0] left, right;
always @* begin
  left = {4{chipsonido}}^16'h8000;
  right = left;
end
```
Si el número de bits que genera el core para las muestras de audio no es divisor entero de 16 bits, por ejemplo, que saque 10 bits de audio mono, y en binario natural, se puede hacer esto (audio es aquí una señal de 10 bits):
```
reg [15:0] left, right;
always @* begin
  left = {audio, audio[9:4]}^16'h8000;
  right = left;
end
```
Es decir, la idea es ir concatenando el valor binario de la señal original tantas veces como sea necesario para alcanzar los 16 bits. Si en la última concatenación "sobran" bits, como en este último caso, pues se concatena sólo el cacho de señal necesario para llegar a los 16 bits. La concatenación es siempre por la derecha. el ^16'h8000 que se pone después hace "la magia" de convertir el valor de 16 bits sin signo a 16 bits complemento a 2.
```
  output reg [5:0] ro,
  output reg [5:0] go,
  output reg [5:0] bo,
  output reg hsync,
  output reg vsync,
  //////////////////////////////////////////
```
Esta parte es sencilla: son las señales RGB de salida (6 bits cada una) y los sincronismos H/V de la salida VGA. Se conectan tal cual, directamente. Por aquí irá, o bien una señal RGB 15 kHz, o bien VGA, según el valor de la señal **video_oputput_sel**
```
  output wire sd_audio_l,
  output wire sd_audio_r,
  output wire i2s_bclk,
  output wire i2s_lrclk,
  output wire i2s_dout,
  //////////////////////////////////////////
```
Esto es la salida de sonido, ya directa al exterior. Las dos primeras señales son la salida sigma-delta heredada del ZXUNO/ZXDOS, y las otras tres van también directas, al exterior de la FPGA, al chip DAC I2S.
```  
  input wire joy_data,
  input wire joy_latch_megadrive,
  output wire joy_clk,
  output wire joy_load_n,
  output wire joy1up,
  output wire joy1down,
  output wire joy1left,
  output wire joy1right,
  output wire joy1fire1,
  output wire joy1fire2,
  output wire joy1fire3,
  output wire joy1start,
  output wire joy2up,
  output wire joy2down,
  output wire joy2left,
  output wire joy2right,
  output wire joy2fire1,
  output wire joy2fire2,
  output wire joy2fire3,
  output wire joy2start,
  //////////////////////////////////////////
```
Todas estas señales corresponden a la gestión de los dos puertos de joystick del ZXTRES:

**joy_data** **joy_clk** y **joy_load_n** se conectan directamente afuera de la FPGA, a los chips serializadores de datos.

**joy_latch_megadrive** la ponemos a 1.

El resto de señales, que son de salida del ZX3W, podemos usarlas como entradas en nuestro core. Cada señal corresponde a un contacto del joystick. Son activas a nivel alto. Las señales FIRE3 y START sólo están disponibles en un pad Megadrive. Algunos pads como Megadrive y Sega Master System tiene FIRE2, y cualquier otro joystick compatible con norma Atari, tiene al menos FIRE1.
```  
  output wire dp_tx_lane_p,
  output wire dp_tx_lane_n,
  input wire dp_refclk_p,
  input wire dp_refclk_n,
  input wire dp_tx_hp_detect,
  inout wire dp_tx_auxch_tx_p,
  inout wire dp_tx_auxch_tx_n,
  inout wire dp_tx_auxch_rx_p,
  inout wire dp_tx_auxch_rx_n,
  ///////////////////////////////////////////
```
Estas señales se conectan directamente afuera de la FPGA, a toda la electrónica que gobierna el DisplayPort. Conectalas, incluso si no planeas usar el DisplayPort para nada, ya que la generación de la VGA depende también de ellas.
```
  output wire dp_ready,
  output wire dp_heartbeat
  );
```
Señales de depuración de la parte DP. Podemos ignorarlas de momento. Las dejamos sin conectar.
```
  parameter [10:0] HSTART = 0;
  parameter [10:0] VSTART = 0;
  parameter [26:0] CLKVIDEO = 25;
  parameter INITIAL_FIELD = 0;
```

## Calculo de los parámetros de ZX3W
Cuando se instancie el ZX3W, hay que especificar cuatro parámetros:
- **CLKVIDEO** indica la frecuencia en MHz del reloj que usamos como dotclock. OJO: es la frecuencia sin tener en cuenta el enable. Por ejemplo, en el Spectrum, CLKVIDEO son 28 MHz, aunque el dotclock son 14 MHz. Pues en ese caso, CLKVIDEO valdría 28, no 14.
- **HSTART** y **VSTART** indican la coordenada horizontal y vertical, medida en píxeles, de la pantalla, donde comenzará a grabarse la información en el framebuffer.
- **INITIAL_FIELD** puede ser 0 o 1, e indica cuál es el campo inicial que se va a mostrar, si el par o el impar. Esto sólo tiene sentido en el A200T, y sólo si al mostrar imagen en VGA o DP, los campos salen desordenados. Si con 0 no funciona, prueba con 1, o viceversa. La mayoría de cores que producen una señal de TV no usan entrelazado, sino progresivo.

A ver: aquí está la madre del cordero. El framebuffer es de 640x240 puntos en A35T y A100T, y de 640x480 en A200T. Puede ser de más, o de menos, según el uso de la pantalla en el core, pero este tamaño propuesto seguramente será bastante estándar. En las FPGAs que usan un framebuffer de 240 líneas, durante la generación de la señal VGA y DP, cada linea se envia repetida dos veces.

Por otra parte, la señal de video original, sobre todo si es PAL, tendrá del orden de 704 - 720 pixeles en horizontal X 288 píxeles en vertical, para cada campo. 576 si fuera PAL completo progresivo, pero los arcades y micros de la época usan PAL progresivo de un solo campo, que son 288 píxeles en PAL y 240 en NTSC.

Entonces, en el framebuffer se grabará una ventana del frame de video original. Un cacho (grande, eso sí) de ese frame. Perderemos video por los bordes. Si os fijáis en el core de Spectrum, cuando se ve por VGA o DP, el borde parece más fino de lo habitual, y es precisamente por eso, porque se pierde borde por los cuatro lados. En el core de test de carta de ajuste se ve mejor lo que se pierde por los bordes (de hecho, para eso está ese core, para que se vea).

Para hilar má fino: **HSTART** usa un contador de 11 bits que comienza a contar desde el momento en que termina el pulso de sincronismo horizontal. Después de que ocurre eso, hay unos cuántos píxeles a negro (front porch) antes de que comience a pintarse imagen.
Por otra parte, **VSTART** hace lo propio pero comienza a contar desde el momento en que termina el pulso de sincronismo vertical.

En el siguiente ejemplo uso como valores 112 y 43. Vamos a ver de dónde vienen estos números:

Partimos de que tenemos que conocer qué tipo de señal de video se genera originalmente en el core. En el ejemplo, uso un generador de sincronismos que implementa el siguiente ModeLine:
```
ModeLine "704x288" 14 704 750 815 896 288 290 293 312
```
Esto es: mi generador de video usa un reloj de 14 MHz, y genera 704 píxeles visibles por scan (valores 0 a 703). Del pixel 704 al 749 hay píxeles negros (back porch). Del 750 al 815 es el sincronismo horizontal, y del 816 al 895 es de nuevo blanking (back porch). En esa última parte hay 80 píxeles (895-816+1).

Esto para la parte de generación de píxeles en cada scan. Según lo dicho, el contador del que depende **HSTART** comienza su cuenta en el pixel 816. De ahí a 895 hay 80 píxeles como ya hemos dicho, y luego hay 704 píxeles de video real.

Quiero que **HSTART** sea tal, que en cada scan, se grabe la parte central del mismo. Es decir, que como no me caben 704 píxeles del scan en el framebuffer, en lugar de grabar cuando HSTART vale 80 (correspondiente al momento en que comienza a generarse video real), comenzaré a grabar un poquito después. ¿Cuándo? Pues a ver: me caben 640 píxeles y tengo 704, así que me sobran 704-640=64 píxeles entre lo que me sobra a izquierda y derecha. A la izquierda, me sobra la mitad: 32. Ese 32 es el poquito de más que tengo que añadir a 80 para determinar dónde grabar: 80+32=**112**.

Ahora lo mismo pero para **VSTART**. Del 'ModeLine' anterior, vemos que se generan 288 lineas de video real, y que en total son 312, contando blanking, sincronismos y demás. El contador de **VSTART** comienza a contar cuando sale del sincronismo vertical, es decir, después de la linea 293. De la 294 a la 312, inclusive, hay 312-294+1=19 lineas. Eso, antes de que comience la primera linea de pantalla, arriba del todo, a dibujarse.

A esas 19 líneas hay que añadir cuántas líneas me voy a saltar (porque no caben). Puedo grabar 240 líneas de las 288 que genero, así que me sobran 288-240=48 líneas contando arriba y abajo. En uno de los lados, arriba, serán 24 líneas. Sumadas a las 19 que tenía, me dan **43**.

Lo que obtengo, en definitiva, es una ventana de video de 640x480 pixeles, en los que descarto 32 píxeles a cada lado izquierda-derecha, y 24 píxeles a cada lado arriba-abajo. En este ejemplo (y en el ZX Spectrum), esta mordida es admisible. En otros cores quizás no, y habrá que hacer un poco encaje de bolillos para poder meter más píxeles en el framebuffer. Por ejemplo, para un core de Amstrad CPC, en horizontal ya tengo que guardar 640 píxeles, y descartar todo el borde. En vertical tengo 200 líneas pero puedo guardar 240, así que tendré algo de borde arriba y abajo, pero nada de borde a izquierda y derecha.

## Ejemplo de instanciación completa

Esta es la instanciación del zx3w en el core de test de carta de ajuste. Este core usa un modo de pantalla PAL de 704x576 lineas, PAL entrelazado, con dos campos de 288 líneas cada uno. La temporización es ligeramente diferente del ejemplo anterior, por lo que los valores para HSTART y VSTART están ligeramente cambiados, como podeis ver.

Cuando os toque portar un core y añadir este módulo, es más conveniente ir poco a poco, conectando cada vez más señales del wrapper al core. En la siguiente sección veremos una posible estrategia de conexión.

```
  localparam HSTART = 110;   // Estos valores centran muy bien una pantalla PAL generada con 
  localparam VSTART = 44;    // timings standard, de 704x576, y con un reloj de 14 MHz
  localparam CLKVIDEO = 14;
  localparam INITIAL_FIELD = 1;   // 1 o 0. Si la fuente no es entrelazada, dejar a 0
  
  zxtres_wrapper #(.HSTART(HSTART), .VSTART(VSTART), .CLKVIDEO(CLKVIDEO), .INITIAL_FIELD(INITIAL_FIELD)) scaler (
  .clkvideo(clkvideo),                    // reloj de pixel de la señal de video original (generada por el core)
  .enclkvideo(1'b1),                      // si el reloj anterior es mayor que el reloj de pixel, y se necesita un clock enable
  .clkpalntsc(clkpalntsc),                // Reloj de 100 MHz para la generacion del reloj de color PAL o NTSC
  .reset_n(clock_stable),                 // Reset de todo el módulo (reset a nivel bajo)
  .reboot_fpga(reset_maestro),            // a 1 para indicar que se quiere volver al core principal
  ///////////////////////////////////////////////////////////////////////////////////////////////////////////
  .video_output_sel(video_output_sel),    // 0: RGB 15kHz + DP   1: VGA + DP pantalla azul
  .disable_scanlines(disable_scanlines),  // 0: emular scanlines (cuidado con el policía del retro!)  1: sin scanlines
  .monochrome_sel(monochrome_sel),        // 0 : RGB, 1: fósforo verde, 2: fósforo ámbar, 3: escala de grises
  .interlaced_image(1'b1),                // Indico que la fuente de video es una señal entrelazada, no progresiva.
  .ad724_modo(1'b0),                      // Reloj de color. 0 : PAL, 1: NTSC
  .ad724_clken(ad724_clken),              // 0 = AD724 usa su propio cristal. 1 = AD724 usa reloj de FPGA.
  ////////////////////////////////////////////////////////////////////////////////////////////////////////////
  .ri(video_r),            //  
  .gi(video_g),            // RGB (8 bits por color primario) de entrada
  .bi(video_b),            // 
  .hsync_ext_n(video_hs),  // Sincronismo horizontal y vertical separados. Los necesito separados para poder, dentro del módulo
  .vsync_ext_n(video_vs),  // medir cuándo comienza y termina un scan y un frame, y así centrar la imagen en el framebuffer
  .csync_ext_n(video_cs),  // entrada de sincronismo compuesto de la señal original
  ////////////////////////////////////////////////////////////////////////////////////////////////////////////
  .audio_l(audio_l),            // Entrada de audio
  .audio_r(audio_r),            // 16 bits, PCM, ca2
  .i2s_bclk(i2s_bclk),          //
  .i2s_lrclk(i2s_lrclk),        // Salida hacia el módulo I2S  
  .i2s_dout(i2s_dout),          //
  .sd_audio_l(audio_out_left),  // Salida de 1 bit desde
  .sd_audio_r(audio_out_right), // los conversores sigma-delta
  ////////////////////////////////////////////////////////////////////////////////////////////////////////////
  .ro(vga_r),         // Salida RGB de VGA 
  .go(vga_g),         // o de 15 kHz, según el valor
  .bo(vga_b),         // de video_output_sel
  .hsync(vga_hs),     // Para RGB 15 kHz, aqui estará el sincronismo compuesto
  .vsync(vga_vs),     // Para RGB 15 kHz, de momento se queda al valor 1, pero aquí luego irá el reloj de color x4
  ////////////////////////////////////////////////////////////////////////////////////////////////////////////
  .dp_tx_lane_p(dp_tx_lane_p),          // De los dos lanes de la Artix 7, solo uso uno.
  .dp_tx_lane_n(dp_tx_lane_n),          // Cada lane es una señal diferencial. Esta es la parte negativa.
  .dp_refclk_p(dp_refclk_p),            // Reloj de referencia para los GPT. Siempre es de 135 MHz
  .dp_refclk_n(dp_refclk_n),            // El reloj también es una señal diferencial.
  .dp_tx_hp_detect(dp_tx_hp_detect),    // Indica que se ha conectado un monitor DP. Arranca todo el proceso de entrenamiento
  .dp_tx_auxch_tx_p(dp_tx_auxch_tx_p),  // Señal LVDS de salida (transmisión)
  .dp_tx_auxch_tx_n(dp_tx_auxch_tx_n),  // del canal AUX. En alta impedancia durante la recepción
  .dp_tx_auxch_rx_p(dp_tx_auxch_rx_p),  // Señal LVDS de entrada (recepción)
  .dp_tx_auxch_rx_n(dp_tx_auxch_rx_n),   // del canal AUX. Siempre en alta impedancia ya que por aquí no se transmite nada.
  /////////////////////////////////////////////////////////////////////////////////////////////////////////////
  .dp_ready(led[0]),
  .dp_heartbeat(led[1])
  );
```
