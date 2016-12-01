---
layout: page
title: "Modelos ARIMA no R"
author: "Regis Augusto Ely"
date: "Maio de 2014 - First draft"
output:
  html_document: default
  pdf_document:
    keep_tex: yes
---

Modelos ARIMA no R
===============================

Regis Augusto Ely

Departamento de Economia

Universidade Federal de Pelotas

Maio de 2014 - Primeira vers�o
------------------------------


Neste exerc�cio iremos ver alguns exemplos de implementa��o dos modelos ARIMA que estudamos em aula. A l�gica � reconstruir a din�mica por tr�s dos modelos e entender melhor os resultados te�ricos que vimos durante o curso de s�ries temporais no PPGOM/UFPel. Por isso vamos simular e estimar os modelos na "m�o", sem utilizar fun��es prontas do R. J� para a realiza��o do diagn�stico e previs�o iremos utilizar alguns fun��es que nos facilitam o c�lculo.

## Simula��o

Vamos simular alguns modelos ARMA no R. Para isso, primeiro devemos definir os par�metros desejados. Voc� pode alterar ou incluir os coeficientes da maneira que quiseres.


```r
# Defini��o dos par�metros
n <- 400  # Total de observa��es simuladas
nx <- 100  # Total de observa��es descartadas
c <- 0.3  # Constante do AR
phi <- c(0.5, -0.8)  # Vetor de par�metros do AR
mu <- 0.7  # M�dia do MA
theta <- c(0.8, 0.3, -0.2)  # Vetor de par�metros do MA
ca <- 0.2  # Constante do ARMA
phia <- c(0.6)  # Vetor de par�metros AR do ARMA
thetaa <- c(0.3)  # Vetor de par�metros MA do ARMA
```


Agora vamos criar os vetores para armazenarmos os dados e o ru�do branco que ser� o componente alet�rio dos modelos, al�m de calcular as ordens p e q dos processos.


```r
# Cria��o dos vetores e c�lculo dos valores de p, d e q.
set.seed(12345)
rb <- rnorm(n)  # Ru�do branco
yar <- numeric(n)  # Vetor do AR
yma <- numeric(n)  # Vetor do MA
yarma <- numeric(n)  # Vetor do ARMA
p <- length(phi)  # Ordem do AR
q <- length(theta)  # Ordem do MA
pa <- length(phia)  # Ordem AR do ARMA
qa <- length(thetaa)  # Ordem MA do ARMA
```


Iremos agora gerar n observa��es das s�ries, descartando as nx primeiras. Primeiro vamos simular um ru�do branco e um passeio aleat�rio. Os gr�ficos s�o plotados no final do comando.



```r
# Simula��o do ru�do branco e do passeio aleat�rio
ruido <- rb[-1:-nx]  # Ru�do Branco
rw <- cumsum(rb)
passeio <- rw[-1:-nx]  # Passeio aleat�rio
par(mfrow = c(1, 2))
ts.plot(ruido, gpars = list(xlab = "Tempo", ylab = "Ru�do Branco"))
ts.plot(passeio, gpars = list(xlab = "Tempo", ylab = "Passeio Aleat�rio"))
```

![plot of chunk unnamed-chunk-3](figure/unnamed-chunk-3.png) 


Agora vamos simular os processos AR, MA e ARMA. O objetivo aqui � construir os processos atrav�s de sua estrutura recursiva, sem utilizar nenhuma fun��o espec�fica do R para isso. Os valores e a quantidade de par�metros dos modelos s�o definidos acima no primeiro bloco de c�digo. No final, plotamos os gr�ficos destes processos.


```r
### Simula��o do processo AR(p)
for (i in (p + 1):n) {
    psum <- 0
    for (j in 1:p) {
        psum <- psum + phi[j] * yar[i - j]
    }
    yar[i] <- psum + c + rb[i]
}
ar <- yar[-1:-nx]

### Simula��o do processo MA(q)
for (i in (q + 1):n) {
    psum <- 0
    for (k in 1:q) {
        psum <- psum + theta[k] * rb[i - k]
    }
    yma[i] <- psum + mu + rb[i]
}
ma <- yma[-1:-nx]

### Simula��o do processo ARMA(p,q)
for (i in (pa + 1):n) {
    psum <- 0
    for (j in 1:pa) {
        psum <- psum + phia[j] * yarma[i - j]
    }
    for (k in 1:qa) {
        psum <- psum + thetaa[k] * rb[i - k]
    }
    yarma[i] <- psum + ca + rb[i]
}
arma <- yarma[-1:-nx]

# Plotando os resultados
par(mfrow = c(1, 3))
ts.plot(ar, gpars = list(xlab = "Tempo", ylab = "AR(p)"))
ts.plot(ma, gpars = list(xlab = "Tempo", ylab = "MA(q)"))
ts.plot(arma, gpars = list(xlab = "Tempo", ylab = "ARMA(p,q)"))
```

![plot of chunk unnamed-chunk-4](figure/unnamed-chunk-4.png) 


Note que estas s�ries parecem apresentar uma persist�ncia maior dos choques do que a s�rie do ru�do branco. � esta persist�ncia que podemos modelar.

Podemos gravar os dados gerados em um arquivo .csv se quisermos


```r
# Gravar dados no arquivo 'simulations.csv'
export <- cbind(ruido, passeio, ar, ma, arma)
write.csv(export, file = "simulations.csv")
```


## Identifica��o

Vamos carregar alguns dados reais para identificarmos qual modelo ARMA melhor se encaixa. Iremos observar uma s�rie dos n�veis do [lago Huron](http://en.wikipedia.org/wiki/Lake_Huron) durante os anos de 1875-1972. Para identificarmos o melhor modelo para esta s�rie devemos olhar para a fun��o de autocorrela��o e a fun��o de autocorrela��o parcial.


```r
data(LakeHuron)
par(mfrow = c(1, 3))
plot(LakeHuron)
acf(LakeHuron)
pacf(LakeHuron)
```

![plot of chunk unnamed-chunk-6](figure/unnamed-chunk-6.png) 


Repare que a fac parece decair exponencialmente at� se tornar estatisticamente insignificante. J� a facp tem dois picos no lag 1 e 2. Pela metodologia de Box-Jenkins, isso nos indica que o melhor para estimar a autocorrela��o desta s�rie � um processo AR(2).

Agora vamos analisar uma s�rie do valor di�rio de fechamento do �ndice de a��es [DAX](http://en.wikipedia.org/wiki/DAX) durante 1991 a 1998, que � composto pelas 30 maiores companhias alem�s da Bolsa de valores de Frankfurt.


```r
data(EuStockMarkets)
par(mfrow = c(1, 2))
plot(EuStockMarkets[, 1])
acf(EuStockMarkets[, 1])
```

![plot of chunk unnamed-chunk-7](figure/unnamed-chunk-7.png) 


Observe que este �ndice tem um comportamento bem err�tico, que lembra o passeio aleat�rio que simulamos anteriormente. A fac n�o decai exponencialmente, sinalizando que a s�rie n�o parece ser estacion�ria.

Vamos agora tirar a primeira diferen�a desta s�rie, mas antes aplicamos o log, pois ent�o ganhamos a interpreta��o de retorno continuamente composto, al�m de estabilizar um pouco a vari�ncia e transformar poss�veis tend�ncias exponenciais em lineares.


```r
dax <- diff(log(EuStockMarkets[, 1]))
par(mfrow = c(1, 3))
plot(dax)
acf(dax)
pacf(dax)
```

![plot of chunk unnamed-chunk-8](figure/unnamed-chunk-8.png) 


Agora temos uma s�rie com o comportamento de um ru�do branco. Parece que n�o h� nenhuma correla��o entre retornos passados e presentes. Bem eficiente este mercado alem�o hein! Parece que nossos modelos ARIMA n�o s�o suficientes para extrair alguma informa��o dos alem�es. Mais tarde veremos alguns modelos n�o lineares que nos dizem algo sobre esses agrupamentos de volatilidade que vemos no primeiro gr�fico acima.

Vamos analisar uma �ltima s�rie temporal da idade da morte de 42 reis sucessivos da Inglaterra. Obteremos estes dados pela internet (uma conex�o � necess�ria). Os dados est�o no site do [Rob J Hyndman](http://robjhyndman.com/), que � professor de estat�stica da Monash University na Austr�lia. Ele tamb�m � autor do pacote forecast no R, que utilizaremos mais a frente. Em seu site h� muito material sobre s�ries e sobre o R, principalmente no t�pico de previs�o, sua especialidade.



```r
kings <- ts(scan("http://robjhyndman.com/tsdldata/misc/kings.dat", skip = 3))
par(mfrow = c(1, 3))
plot(kings)
acf(kings)
pacf(kings)
```

![plot of chunk unnamed-chunk-9](figure/unnamed-chunk-9.png) 


Repare que apesar de parecer estar tudo ok com a fac e facp, a m�dia da s�rie claramente mudou longo do tempo. Isso � um indicativo de n�o estacionariedade. Aqui iremos avan�ar um pouco no conte�do e introduzir o teste de Dickey-Fuller aumentado para verificar essa nossa hip�tese de n�o estacionariedade. Para isso precisaremos instalar um pacote novo no R, chamado [tseries](http://cran.r-project.org/web/packages/tseries/index.html), que cont�m a fun��o adf.test.


```r
if (!require(tseries)) {
    install.packages("tseries", repos = "http://cran.rstudio.com/")
}
library(tseries)
adf.test(kings)
```

```
## 
## 	Augmented Dickey-Fuller Test
## 
## data:  kings
## Dickey-Fuller = -2.113, Lag order = 3, p-value = 0.529
## alternative hypothesis: stationary
```


Aceitamos a hip�tese de uma ra�z unit�ria, o que nos diz que a s�rie n�o � estacion�ria (veremos o teste ADF com cuidado mais a frente). Vamos ent�o tirar a primeira diferen�a e ver o que acontece.


```r
kingsdiff <- diff(kings, 1)
adf.test(kingsdiff)
```

```
## 
## 	Augmented Dickey-Fuller Test
## 
## data:  kingsdiff
## Dickey-Fuller = -4.075, Lag order = 3, p-value = 0.01654
## alternative hypothesis: stationary
```


Agora sim. Rejeitamos a hip�tese nula e parece que a s�rie � estacion�ria. Vamos olhar para os gr�ficos.


```r
par(mfrow = c(1, 3))
plot(kingsdiff)
acf(kingsdiff)
pacf(kingsdiff)
```

![plot of chunk unnamed-chunk-12](figure/unnamed-chunk-12.png) 


Esta s�rie est� um pouco complicada. Temos algumas possibilidades diferentes. Poderia ser um AR(3) ou um MA(1). Vamos com o MA(1) por ser mais parcim�nio (menos coeficientes).

Agora que aprendemos a din�mica dos modelos ARIMA e identificamos nossas s�ries, vamos aprender a estimar os modelos.

## Estima��o

Aqui o objetivo � construir a estimativa de m�xima verossimilhan�a conforme vimos em aula. Para isso teremos que criar uma fun��o para calcular a log-verossimilhan�a e ent�o carreg�-la dentro da fun��o [optim](http://stat.ethz.ch/R-manual/R-devel/library/stats/html/optim.html) do R, que utiliza m�todos de otimiza��o num�rica para achar o valor dos par�metros que minimiza o negativa da fun��o. Podemos ent�o comparar os nossos resultados com a fun�ao [arima](http://stat.ethz.ch/R-manual/R-devel/library/stats/html/arima.html) do R, que estima tudo isso automaticamente.

Professor, mas se a fun��o arima j� faz automaticamente, para que construir a log-verossimilhan�a? Para aprender! O dia que voc� quiser estimar um modelo recente, que n�o haja pacote pronto no R, ou at� um modelo que voc� desenvolveu, ent�o voc� ir� lembrar desse exerc�cio.

Vamos l�! Primeiro iremos estimar o MA(1) que identificamos na s�rie da idade da morte de reis da Inglaterra. A primeira etapa � carregar uma fun��o que nos d� a log-verossimilhan�a de um MA(1) para um conjunto de dados e par�metros. Utilizaremos a fun��o de log-verossimilhan�a condicional, conforme vimos em aula:

$$L(\theta; Y_t) = - \frac{T}{2} \ln(2 \pi) - \frac{T}{2} \ln(\sigma^2) - \sum_{t=1}^T \frac{\varepsilon_t^2}{2 \sigma^2}$$

E lembrem que $\varepsilon_t = y_t - \mu - \theta \varepsilon_{t-1}$ no caso de um MA(1). Iremos tamb�m supor que $\varepsilon_0 = 0$. Ent�o vamos construir essa fun��o.


```r
loglik.ma <- function(param, data) {
    T <- length(data)
    res <- numeric(T)
    res[1] <- data[1] - param[1]
    c1 <- -(T/2) * log(2 * pi)
    for (i in 2:T) {
        res[i] <- data[i] - param[1] - param[2] * res[i - 1]
    }
    c2 <- -(T/2) * log(param[3])
    c3 <- -sum((res^2)/(2 * param[3]))
    L <- c1 + c2 + c3
    -L
}
```


Agora vamos chutar par�metros iniciais e ent�o mandar a fun��o optim encontrar os par�metros que minimizam essa fun�ao acima. Note que os nossos par�metros s�o $(\mu, \theta, \sigma^2)$. A fun��o ir� retornar alguns warnings pelo fato de tentar valores negativos para $\sigma^2$, mas por enquanto isso n�o � problema para n�s. Tamb�m podemos calcular os erros dos coeficientes pegando os elementos da diagonal da ra�z quadrada da matriz inversa da hessiana. Comparamos os resultados com a fun��o arima do R.


```r
# Dando um chute inicial e estimando o MA(1)
param_ma <- c(0.5, 0.1, 10)
estma <- optim(par = param_ma, fn = loglik.ma, method = "BFGS", hessian = TRUE, data = kingsdiff)

# Calculando o erro padr�o dos coeficiente
estma_se <- sqrt(diag(solve(estma$hessian)))

# Estimando pela fun��o arima e comparando os resultados numa matriz
estma_arima <- arima(kingsdiff, order = c(0,0,1))
result_ma <- matrix(c(estma_arima$coef[2:1], estma_arima$sigma2, sqrt(diag(estma_arima$var.coef))[2:1], NA), 2, 3, byrow=TRUE)
ma_comp <- rbind(estma$par, result_ma[1,], estma_se, result_ma[2,])
colnames(ma_comp) <- c("mu", "theta", "sigma2")
rownames(ma_comp) <- c("optim.coef", "arima.coef", "optim.se", "arima.se")
ma_comp
```

```
##                mu   theta sigma2
## optim.coef 0.3292 -0.7417 209.95
## arima.coef 0.3882 -0.7463 228.24
## optim.se   0.6199  0.1105  42.35
## arima.se   0.6522  0.1278     NA
```


Os resultados s�o parecidos n�o? Alguma diferen�a existe, pois o m�todo da fun��o arima utiliza como chutes iniciais a estimativa de min�mos quadr�ticos (que � um chute inicial melhor do que o nosso). Note que a fun��o arima n�o calcula a vari�ncia do erro, por isso o NA no �ltimo elemento.

Agora vamos estimar um modelo AR(1) utilizando a express�o da log-verossimilhan�a que vimos na aula:

$$ L(\phi, Y_t) = - \left[ \frac{T-1}{2} \right] \ln(2 \pi) - \left[ \frac{T-1}{2} \right] \ln(\sigma^2) - \sum_{t=2}^T \frac{\varepsilon_t^2}{2 \sigma^2} $$

Lembrando que agora $\varepsilon_t = y_t - c - \phi y_{t-1}$. Vamos utilizar a s�rie . Seguiremos as mesmas etapas, primeiro carregando a fun��o de log-verossimilhan�a. Note que teremos de novo 3 par�metros $(c, \phi, \sigma^2)$, e iremos supor que $y_0 = 0$.


```r
loglik.ar <- function(param, data) {
    T <- length(data)
    res <- numeric(T)
    res[1] <- data[1] - param[1]
    c1 <- -((T - 1)/2) * log(2 * pi)
    for (i in 2:T) {
        res[i] <- data[i] - param[1] - param[2] * data[i - 1]
    }
    c2 <- -((T - 1)/2) * log(param[3])
    c3 <- -sum((res^2)/(2 * param[3]))
    L <- c1 + c2 + c3
    -L
}
```


Para realizar a estima��o, iremos gerar uma s�rie AR(1) com valores $c = 0.7$, $\phi = 0.8$ e $\sigma^2 = 1$. Ent�o comparamos os resultados com a fun��o arima do R.


```r
# Criando um AR(1)
set.seed(12345)
yt <- numeric(1000)
eps <- rnorm(1000)
c <- 0.7
phi <- 0.8
for (i in 2:1000) {
    yt[i] <- c + phi * yt[i - 1] + eps[i]
}

# Dando um chute inicial e estimando o AR(1)
param_ar <- c(1, 0.5, 0.5)
estar <- optim(par = param_ar, fn = loglik.ar, method = "BFGS", hessian = TRUE, 
    data = yt)

# Calculando o erro padr�o dos coeficiente
estar_se <- sqrt(diag(solve(estar$hessian)))

# Estimando pela fun��o arima e comparando os resultados numa matriz
estar_arima <- arima(yt, order = c(1, 0, 0))
result_ar <- matrix(c(estar_arima$coef[2:1], estar_arima$sigma2, sqrt(diag(estar_arima$var.coef))[2:1], 
    NA), 2, 3, byrow = TRUE)
ar_comp <- rbind(estar$par, result_ar[1, ], estar_se, result_ar[2, ])
colnames(ar_comp) <- c("c", "phi", "sigma2")
rownames(ar_comp) <- c("optim.coef", "arima.coef", "optim.se", "arima.se")
ar_comp
```

```
##                  c     phi  sigma2
## optim.coef 0.70172 0.81163 0.99735
## arima.coef 3.69315 0.81376 1.00055
## optim.se   0.07517 0.01837 0.04462
## arima.se   0.16912 0.01843      NA
```


Os resultados s�o semelhantes, com exce��o do coeficiente c. Isso porque a fun��o arima calcula o intercepto como sendo $\mu$ e n�o c. Mas lembre que $\mu = \frac{c}{1-\phi}$. Assim, podemos calcular o c do modelo estimado pela fun��o arima e comparar com o nosso.


```r
ar_comp[2, 1] * (1 - ar_comp[2, 2])
```

```
## [1] 0.6878
```


Agora sim, o resultado est� pr�ximo. Est� na hora de vermos se a nossa estima��o est� boa, ou seja, se estamos conseguindo remover completamente a correla��o dos res�duos.

## Diagn�stico

Para verificarmos se o modelo est� bem especificado, devemos olhar para os res�duos da estima��o.


```r
par(mfrow = c(1, 2))
plot(estma_arima$residuals, ylab = "Res�duos do MA(1)")
plot(estar_arima$residuals, ylab = "Res�duos do AR(1)")
```

![plot of chunk unnamed-chunk-18](figure/unnamed-chunk-18.png) 


Parecem estacion�rios, mas para sabermos se eliminamos completamente a autocorrela��o residual devemos fazer o teste de Ljung-Box.


```r
Box.test(estma_arima$residuals, lag = 15, type = "Ljung-Box")
```

```
## 
## 	Box-Ljung test
## 
## data:  estma_arima$residuals
## X-squared = 10.14, df = 15, p-value = 0.8107
```

```r
Box.test(estar_arima$residuals, lag = 15, type = "Ljung-Box")
```

```
## 
## 	Box-Ljung test
## 
## data:  estar_arima$residuals
## X-squared = 15.6, df = 15, p-value = 0.4092
```


Ok, aceitamos a hip�tese nula de que os res�duos n�o s�o autocorrelacionados. Agora temos um bom modelo linear para descrever nossos dados. Vamos realizar alguma previs�o para estas vari�veis.

## Previs�o

O melhor pacote de previs�o do R � o [forecast](http://cran.r-project.org/web/packages/forecast/index.html). Irei carregar ele e ent�o realizar uma previs�o de 10 horizonte para o MA e o AR. Este pacote utiliza uma fun��o interna para a estima��o chamada Arima. Antes de fazer a previs�o vamos estimar os modelos de novo atrav�s dessa fun��o.



```r
if (!require(forecast)) {
    install.packages("forecast", repos = "http://cran.rstudio.com/")
}
library(forecast)
Ma <- Arima(kingsdiff, order = c(0, 0, 1))
Ar <- Arima(yt, order = c(1, 0, 0))

prev_ma <- forecast.Arima(Ma, h = 10)
prev_ma
```

```
##    Point Forecast   Lo 80 Hi 80  Lo 95 Hi 95
## 43        13.3630  -5.998 32.72 -16.25 42.97
## 44         0.3882 -23.770 24.55 -36.56 37.34
## 45         0.3882 -23.770 24.55 -36.56 37.34
## 46         0.3882 -23.770 24.55 -36.56 37.34
## 47         0.3882 -23.770 24.55 -36.56 37.34
## 48         0.3882 -23.770 24.55 -36.56 37.34
## 49         0.3882 -23.770 24.55 -36.56 37.34
## 50         0.3882 -23.770 24.55 -36.56 37.34
## 51         0.3882 -23.770 24.55 -36.56 37.34
## 52         0.3882 -23.770 24.55 -36.56 37.34
```

```r
prev_ar <- forecast.Arima(Ar, h = 10)
prev_ar
```

```
##      Point Forecast Lo 80 Hi 80   Lo 95 Hi 95
## 1001          2.589 1.307 3.871 0.62866 4.550
## 1002          2.795 1.142 4.447 0.26718 5.322
## 1003          2.962 1.104 4.820 0.12058 5.804
## 1004          3.098 1.116 5.080 0.06670 6.130
## 1005          3.209 1.149 5.269 0.05796 6.360
## 1006          3.299 1.189 5.410 0.07140 6.527
## 1007          3.373 1.229 5.516 0.09495 6.650
## 1008          3.432 1.268 5.597 0.12208 6.742
## 1009          3.481 1.302 5.659 0.14927 6.812
## 1010          3.520 1.333 5.708 0.17471 6.866
```

```r

par(mfrow = c(1, 2))
plot(prev_ar, include = 100)
plot(prev_ma, include = 100)
```

![plot of chunk unnamed-chunk-20](figure/unnamed-chunk-20.png) 


Note que no MA(1), a previs�o converge para a m�dia ap�s o primeiro per�odo, enquanto que no AR, a previs�o ir� lentamente convergir para a m�dia. Exatamente como vimos em aula quando calculamos isso! :)
