Costos del café colombiano
================
22 de julio de 2016

En este documento hacemos un análisis breve de los precios del café colombiano y sus costos de producción. El artículo explicativo se encuentra en mi blog [Vasija de Ideas](http://vasijadeideas.blogspot.com/2016/07/esta-gente-paga-para-que-la-dejen.html). El software utilizado es R 3.2.2.

Investigador: Norman Simón Rodríguez, MSc.

Los datos
=========

Los datos provienen de diversas fuentes:

1.  **Precio internacional del café:** La información fue obtenida de la página web de la Organizacién Internacional del Café (ICO, por sus siglas en inglés), y corresponde a los precios históricos de referencia para los cafés tipo suave colombiano (Colombian Milds). La tabla en Excel puede ser descargada desde <http://www.ico.org/new_historical.asp>, y corresponde a la categoría "ICO Composite & Group Indicator Prices - Monthly Averages". La unidad es centavos de dólar por libra de 453.6 gramos (promedios mensuales). Nombre del archivo: "ico.csv".

2.  **Precio interno del café en Colombia:** Se descargó una tabla de Excel de la página de la Federación Nacional de Cafeteros (<http://www.federaciondecafeteros.org/particulares/es/quienes_somos/119_estadisticas_historicas/>), correspondiente a la categoría "3. Precio interno base - mensual desde 1944". Este precio *aparentemente no* incluye los subsidios gubernamentales al precio la AIC y el PIC (véase documentos "Precio café 2013.pdf" en la carpeta "data" y la captura de pantalla de <https://pic.federaciondecafeteros.org/> llamada "PIC 2014.pdf"). Este es el precio interno de referencia que paga la Federación a los cafeteros. La unidad es pesos colombianos por carga de 125 kilogramos (promedios mensuales). Nombre del archivo: "fedecafe.csv".

3.  **Índice de precios al productor:** Con este indicador podemos estimar los costos de producción de los caficultores para cada mes de la serie de tiempo. Fue obtenido de la página web del DANE: <http://www.dane.gov.co/index.php/precios-e-inflacion/indice-de-precios-al-productor>, categoría "Históricos". Este índice corresponde al indicador de precios al productor para producción nacional en agricultura, ganadería, caza, silvicultura y pesca. Base diciembre 2014=100. Nombre del archivo "ipp.csv".

4.  **Tasa de cambio:** Obtenida de la página del Banco de la República (<http://www.banrep.gov.co/es/trm>). Unidad: pesos por dólar estadounidense, promedio mensual. Nombre del archivo: "tcm.csv".

5.  **Costo promedio de producción por carga de café:** Este dato se obtuvo a partir de un derecho de petición respondido por la Federación Nacional de Cafeteros en el año 2013. Corresponde a 654459 pesos por carga de 125 kilogramos para el año 2012. Datos disponibles sobre demanda.

Procesamiento
=============

En primer lugar, debemos importar los datos y ponerlos todos en el mismo dataframe:

``` r
ico <- read.csv("./data/ico.csv", dec=",")
ico$date <- paste(ico$year,"-",match(ico$month,month.name), "-15", sep="")
ico$date <- as.Date(ico$date)

tcm <- read.csv("./data/tcm.csv", dec=",")
tcm$date <- paste(tcm$year,"-",match(tcm$month,month.name), "-15", sep="")
tcm$date <- as.Date(tcm$date)

datos <- merge(tcm, ico, by="date")

fedecafe <- read.csv("./data/fedecafe.csv", dec=".")
fedecafe$date <- paste(fedecafe$year,"-",match(fedecafe$month,month.name), "-15", sep="")
fedecafe$date <- as.Date(fedecafe$date)

datos <- merge(datos, fedecafe, by="date")

ipp <- read.csv("./data/ipp.csv", dec=",")
ipp$date <- paste(ipp$year,"-",match(ipp$month,month.name), "-15", sep="")
ipp$date <- as.Date(ipp$date)

datos <- merge(datos, ipp, by="date")


datos <- datos[,c("date", "exchangerate", "colombianmilds", "agricultureipp", "internalprice")]
```

Ahora convertimos los datos de ICO a pesos colombianos:

``` r
icopc <- (as.numeric(datos$colombianmilds)*as.numeric(datos$exchangerate))/100
```

Y luego a kilogramos

``` r
icopc <- (icopc/453.6)*1000
```

Asimismo, convertimos los datos de precio interno de cargas a kilos:

``` r
federak <- datos$internalprice/125
```

Así que ya tenemos los precios internos por kilo y los precios internacionales por kilo. Ahora vamos a hallar el costo de producción por carga (deflactamos tomando diciembre de 2012 como nuevo año base de nuestro IPP, ya que el dato de costos lo tenemos para el 2012):

``` r
costo2012 <- 654459

costokilo <- datos$agricultureipp*costo2012/datos$agricultureipp[which(datos$date=="2012-12-15")]
```

Ahora convertimos a costo por kilo:

``` r
costokilo <- costokilo/125
```

Juntamos los datos calculados en un dataframe:

``` r
datos$icopc <- icopc
datos$federak <- federak
datos$costokilo <- costokilo

datosnuevos <- datos[,c("date", "icopc", "federak", "costokilo")]
datosnuevos$date <- as.Date(datosnuevos$date)
```

Gráfico
=======

Y ahora graficamos los datos:

``` r
library(reshape)
datosnuevos <- melt(datosnuevos, id="date")

library(ggplot2)
library(ggthemes)
p <- ggplot(datosnuevos, aes(x=weight)) + theme_hc() + scale_fill_manual(values=c("red", "green", "white") ,breaks=c("costokilo", "federak", "icopc"), name="", labels=c("Costo por kilo", "Precio interno", "Precio internacional"))

                                                                        
p + geom_area(aes(y=value, x=date, fill=variable), color="black", position = 'identity', alpha = 0.5) +
labs(x="Fecha", y="Pesos colombianos por kilo") + 
  ggtitle("Evolución de precios y costos del café suave colombiano\n Fuente: Vasija de Ideas (para fuentes de datos ver artículo)")
```

![](README_files/figure-markdown_github/unnamed-chunk-8-1.png)
