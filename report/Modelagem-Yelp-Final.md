Sistema de Recomendação de Estabelecimentos em Toronto
================
Marcelo Franceschini, Rafael Costa, Viviane Sanchez
6/27/2020 - 7/12/2020

# 1\. Introdução

Foram selecinados dados de usuários e estabelecimentos abertos em
Toronto para treinar um modelo de classificação binária das notas das
avaliações e assim *prever se a nota de um estabelecimento que o usuário
ainda não avaliou seria boa ou ruim*. A recomendação é dada conforme a
maior probabilidade de o usuário dar uma nota alta para aquele lugar.

## 1.1. Objetivo

**Criar um sistema de recomendação de estabelecimentos bem avaliados em
linha com o perfil do usuário.**

Os dados das seguintes bases fornecidas pelp Yelp para estudos
acadêmicos serão analisados

  - Business - informações e atributos dos estabelecimentos

  - Users - Informações sobre o perfil do usuário no Yelp

  - Reviews - Texto e atributos de avaliações dos estabelecimentos
    criadas pelos usuários

  - Tips - Dicas adicionais e atributos deixados pelos usuários

  - Check-ins - Histórico de check-ins por estabelecimento

  - [Fonte de dados](https://www.yelp.com/dataset)

## 1.2. Tratamento da base

As bases mencionadas foram previamente tratadas em Python/PySpark. Neste
relatório, serão analisadas individualmente as informações dos
estabelecimentos e de usuários. Além disso, uma base contendo os dados
dessas duas bases e das dicas por estabelecimento e usuário foram
consolidados em uma base única para modelagem com uma rede neural. Por
fim, um usuário fictício é criado para que uma recomendação seja feita.

# 2\. Análise Exploratória

## 2.1. Pacotes

``` r
library(tidyverse)
library(tidymodels)
library(tidytext)
library(skimr)
library(ggrepel)
library(factoextra)
library(vip)
library(cluster)
library(plotly)
library(zoo)
library(keras)
library(reticulate)
library(ggmap)
library(rpart)
library(partykit)
library(clue)

#rmarkdown::render("Modelagem Yelp Final.Rmd", envir=.GlobalEnv)
```

## 2.2. Leitura de arquivos individuais

``` r
yelp_bz_raw <- list.files(path = 'output/yelp_bz.csv/', 
                       pattern = "*.csv",
                       full.names = TRUE) %>% 
            map_df(~read_csv(.))

yelp_users <- list.files(path = 'output/yelp_usr.csv/', 
                       pattern = "*.csv",
                       full.names = TRUE) %>% 
            map_df(~read_csv(.))
```

## 2.3. Business

A seguir, será feita uma análise de componentes principais para entender
a variabilidade da nota dos estabelcimentos conforme seus atributos, que
foram codificados numericamente:

  - Atributos com True/False como resposta:
    
      - Null/None: 0
      - False: 1
      - True: 2

  - Atributos que possuem uma descrição das características foram
    substituídas por números como o exemplo:
    
      - Null/None: 0
      - Característica 1 - 1
      - Característica N - N

Em seguida, são removidas as colunas não numéricas (business\_id,
categories) para a análise e, caso exista algum valor faltando, é
substituído por zero.

``` r
yelp_bz <- yelp_bz_raw %>% 
          select_if(~is.numeric(.)) %>% 
          mutate_all(~replace(., is.na(.), 0))

glimpse(yelp_bz)
```

    ## Rows: 14,962
    ## Columns: 34
    ## $ latitude                   <dbl> 43.62661, 43.64041, 43.61129, 43.70441, 43…
    ## $ longitude                  <dbl> -79.50209, -79.39058, -79.55687, -79.37511…
    ## $ review_count               <dbl> 4, 81, 3, 3, 4, 6, 10, 52, 14, 4, 4, 11, 7…
    ## $ stars                      <dbl> 2.0, 2.5, 1.0, 5.0, 3.0, 4.5, 3.0, 2.5, 3.…
    ## $ AcceptsInsurance           <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ AgesAllowed                <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ Alcohol                    <dbl> 0, 2, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ BYOB                       <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ BikeParking                <dbl> 2, 1, 0, 0, 0, 0, 2, 0, 2, 0, 0, 2, 2, 0, …
    ## $ BusinessAcceptsCreditCards <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ ByAppointmentOnly          <dbl> 2, 0, 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ Caters                     <dbl> 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 2, 0, …
    ## $ CoatCheck                  <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ Corkage                    <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, …
    ## $ DogsAllowed                <dbl> 0, 0, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 0, 0, …
    ## $ DriveThru                  <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ GoodForDancing             <dbl> 0, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ GoodForKids                <dbl> 2, 2, 0, 0, 0, 0, 0, 2, 0, 0, 0, 0, 2, 0, …
    ## $ HappyHour                  <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ HasTV                      <dbl> 0, 2, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, …
    ## $ NoiseLevel                 <dbl> 0, 4, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ OutdoorSeating             <dbl> 0, 2, 0, 0, 0, 0, 2, 1, 2, 0, 0, 2, 1, 1, …
    ## $ RestaurantsAttire          <dbl> 0, 3, 0, 0, 0, 0, 3, 3, 3, 0, 0, 3, 3, 0, …
    ## $ RestaurantsDelivery        <dbl> 0, 1, 0, 0, 0, 0, 0, 1, 0, 0, 0, 0, 1, 0, …
    ## $ RestaurantsGoodForGroups   <dbl> 0, 2, 0, 0, 0, 0, 0, 2, 0, 0, 0, 0, 2, 0, …
    ## $ RestaurantsPriceRange2     <dbl> 2, 2, 0, 0, 2, 0, 2, 1, 2, 0, 0, 1, 2, 0, …
    ## $ RestaurantsReservations    <dbl> 0, 2, 0, 0, 0, 0, 0, 2, 0, 0, 0, 0, 1, 0, …
    ## $ RestaurantsTableService    <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 1, 0, …
    ## $ RestaurantsTakeOut         <dbl> 0, 2, 0, 0, 0, 0, 2, 2, 2, 0, 0, 2, 2, 0, …
    ## $ Smoking                    <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ WheelchairAccessible       <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 2, 2, 0, …
    ## $ WiFi                       <dbl> 0, 3, 0, 0, 0, 0, 3, 3, 3, 0, 0, 3, 3, 0, …
    ## $ tips_counter_bz            <dbl> 0, 14, 0, 0, 0, 1, 4, 5, 5, 0, 0, 6, 0, 1,…
    ## $ total_compliments_bz       <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …

### 2.3.1. PCA

A análise PCA é feita com a criação de uma receita pelo pacote
tidymodels (`step_pca`). A base processada é extraída com a função
`juice`.

``` r
rec_pca <- recipe(stars ~ ., yelp_bz) %>% 
  update_role(contains('id'), new_role = 'id') %>% 
  #step_date(date_rv, yelping_since_usr, features = c("dow", "month","year")) %>% 
  #step_other(categories, threshold = 0.005) %>% 
  #step_other(postal_code, threshold = 0.01) %>% 
  #step_dummy(all_nominal(), -'business_id',-'user_id',-'name_bz') %>%
  step_normalize(all_numeric(), -all_outcomes()) %>% 
  step_pca(all_numeric(), -all_outcomes()) %>% 
  step_naomit(all_numeric()) %>% 
  prep()

yelp_bz_pca <- juice(rec_pca)
```

#### 2.3.1.1. Scree plot

``` r
variance_pct <- rec_pca$steps[[2]]$res

(cumsum(variance_pct$sdev^2) / sum(variance_pct$sdev^2))
```

    ##  [1] 0.2795283 0.3569530 0.4181779 0.4760438 0.5246121 0.5650128 0.6019585
    ##  [8] 0.6384455 0.6714870 0.7020952 0.7307504 0.7590915 0.7806310 0.8013364
    ## [15] 0.8214466 0.8408425 0.8588443 0.8747742 0.8894284 0.9032762 0.9169327
    ## [22] 0.9301989 0.9408237 0.9507912 0.9598340 0.9678033 0.9751966 0.9818828
    ## [29] 0.9879589 0.9930369 0.9974065 1.0000000 1.0000000

``` r
fviz_eig(variance_pct, addlabels = TRUE) + 
  labs(x = "Componente Principal",
       y = "Percentual explicado da variância")
```

<img src="Modelagem-Yelp-Final_files/figure-gfm/unnamed-chunk-5-1.png" width="960" />

Nota-se que mais de 50% da variabilidade é explicada pelas 5 primeiras
componentes. Suas composições serão avaliadas no próximo item.

#### 2.3.1.2. Drivers

``` r
tidy_pca <- tidy(rec_pca, 2)

tidy_pca %>%
  filter(component %in% paste0("PC", 1:6)) %>%
  group_by(component) %>%
  top_n(15, abs(value)) %>%
  ungroup() %>%
  mutate(terms = reorder_within(terms, abs(value), component)) %>%
  ggplot(aes(abs(value), terms, fill = value > 0)) +
  geom_col() +
  facet_wrap(~component, scales = "free_y") +
  scale_y_reordered() +
  labs(
    x = "Valor absoluto da contribuição",
    y = NULL, fill = "Valor > 0")
```

<img src="Modelagem-Yelp-Final_files/figure-gfm/unnamed-chunk-6-1.png" width="960" />

Na primeira componente, os pesos são igualmente distribuídos, o que
indica que todos os atributos tem impacto semelhante na maior parte da
variabilidade.

Pela segunda componente, no entanto, observa-se que a existência de um
local para deixar o casaco, ser permitido fumar e ser um bom local para
dançar são mais relevantes, assim como a localização (PC6). Além disso,
cobrança de rolha e necessidade de levar a bebida também são drivers
importantes, pois aparecem em mais de uma componente. Importante notar
que o nível de preço do estabelecimento aparece apenas na 5a componente.

#### 2.3.1.3.Contrastes

Observa-se a interação entre as principais variáveis das duas primeiras
compomentes:

``` r
variance_pct %>% 
  fviz_pca_var(axes = c(1,2), col.var="contrib", gradient.cols = c("#00AFBB", "#E7B800", "#FC4E07"))
```

<img src="Modelagem-Yelp-Final_files/figure-gfm/unnamed-chunk-7-1.png" width="960" />

Os maiores contrastes são entre a modalidade de atendimento dos
restaurante: Apenas delivery ou com reservas.

## 2.4. Users

Para fazer recomendações baseadas no perfil do usuário, é necessários
agrupá-los de alguma forma. Foram testados os algoritmos de
clusterização hierárquica e k-means, mas apenas o último trouxe
resultados interpretáveis e satisfatórios.

### 2.4.1. K-Means

Para definir o número de clusters ideal, é feita uma análise de
sensibilidade e aplicado o [algoritmo kmeans no formato
tidy](\(https://www.tidymodels.org/learn/statistics/k-means/\))

``` r
set.seed(123)

glimpse(yelp_users)
```

    ## Rows: 119,792
    ## Columns: 23
    ## $ user_id            <chr> "-4Anvj46CWf57KWI9UQDLg", "-BUamlG3H-7yqpAl1p-msw"…
    ## $ average_stars      <dbl> 3.50, 1.50, 3.00, 3.56, 3.00, 4.00, 4.17, 3.57, 4.…
    ## $ compliment_cool    <dbl> 0, 0, 0, 0, 0, 0, 0, 169, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ compliment_cute    <dbl> 0, 0, 0, 0, 0, 0, 0, 2, 0, 0, 0, 0, 0, 0, 0, 0, 0,…
    ## $ compliment_funny   <dbl> 0, 0, 0, 0, 0, 0, 0, 169, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ compliment_hot     <dbl> 0, 0, 0, 0, 0, 0, 0, 94, 0, 0, 0, 0, 0, 0, 0, 2, 0…
    ## $ compliment_list    <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,…
    ## $ compliment_more    <dbl> 0, 0, 0, 0, 0, 0, 0, 3, 0, 0, 0, 0, 0, 0, 0, 0, 0,…
    ## $ compliment_note    <dbl> 0, 0, 1, 0, 0, 0, 0, 16, 0, 1, 0, 0, 0, 0, 0, 1, 0…
    ## $ compliment_photos  <dbl> 0, 0, 0, 0, 0, 0, 0, 97, 0, 0, 0, 0, 0, 0, 0, 0, 0…
    ## $ compliment_plain   <dbl> 0, 0, 0, 0, 0, 0, 0, 66, 0, 0, 0, 1, 0, 0, 0, 0, 0…
    ## $ compliment_profile <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,…
    ## $ compliment_writer  <dbl> 0, 0, 0, 0, 0, 0, 0, 30, 0, 0, 0, 0, 0, 0, 0, 0, 0…
    ## $ cool               <dbl> 2, 0, 1, 0, 1, 0, 0, 1562, 2, 1, 1, 9, 0, 5, 0, 9,…
    ## $ elite_count        <dbl> 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1,…
    ## $ fans               <dbl> 1, 0, 0, 0, 0, 0, 0, 39, 0, 0, 0, 1, 0, 0, 0, 0, 0…
    ## $ friends_count      <dbl> 1, 16, 15, 27, 1, 1, 1, 338, 59, 6, 10, 100, 8, 1,…
    ## $ funny              <dbl> 0, 0, 1, 0, 0, 0, 0, 1266, 3, 1, 4, 0, 1, 1, 1, 5,…
    ## $ review_count_usr   <dbl> 2, 2, 4, 27, 2, 6, 6, 66, 28, 3, 8, 37, 4, 20, 1, …
    ## $ useful             <dbl> 2, 0, 1, 5, 1, 3, 16, 1683, 12, 1, 2, 30, 4, 30, 0…
    ## $ year_since         <dbl> 2016, 2016, 2011, 2019, 2014, 2017, 2014, 2019, 20…
    ## $ tips_counter       <dbl> 0, 1, 0, 0, 0, 1, 0, 0, 0, 19, 0, 0, 0, 0, 0, 2, 0…
    ## $ total_compliments  <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,…

``` r
yelp_pad <- yelp_users %>% 
              select(-user_id) %>% 
              scale()

kclusts <- tibble(k = 1:30) %>%
  mutate(kclust = map(k, ~kmeans(yelp_pad, .x)),
        tidied = map(kclust, tidy),
        glanced = map(kclust, glance),
        augmented = map(kclust, augment, yelp_pad)
        )

clusters <- kclusts %>%
  unnest(cols = c(tidied))

assignments <- kclusts %>% 
  unnest(cols = c(augmented))

clusterings <- kclusts %>%
  unnest(cols = c(glanced))
```

``` r
### Cotovelo

clusterings %>% 
  ggplot(aes(k, tot.withinss)) + 
    geom_point(size = 3) + 
    geom_line() + 
    labs(y = "total within sum of squares", x = "k") +
    scale_x_continuous(breaks = 1:30)
```

<img src="Modelagem-Yelp-Final_files/figure-gfm/unnamed-chunk-9-1.png" width="960" />

Pelo gráfico do cotovelo, poderiam ser selecionado um número de clusters
(k) de 11 a 17. a seguir é possível ver a eveolução do algoritmo de
clusterização em relação às dicas deixadas por usuário e o quanto foram
elogiadas pela comunidade.

``` r
#k-means
assignments %>% 
  filter(k %in% paste0(10:20)) %>%
  ggplot(aes(x = tips_counter, y = total_compliments)) +
  geom_point(aes(color = .cluster), alpha = 0.5) + 
  facet_wrap(~ k, nrow = 3)
```

<img src="Modelagem-Yelp-Final_files/figure-gfm/unnamed-chunk-10-1.png" width="960" />

Para a classificação final dos usuário, será feita novamente a
clusterização, mas considerando apenas o número de clusters `k` ideal.

``` r
set.seed(123)
kmeans_usr <-  kmeans(yelp_pad, 11)

yelp_usr_cluster <- yelp_users %>% 
          mutate(cluster_usr = kmeans_usr$cluster)

#glimpse(yelp_usr_cluster)
```

Por fim, observa-se claramente a divisão dos usuários em relação ao
tempo na plataforma e a nota média do usuário. Além disso, nota-se
diferentes camadas em relação o número de fãs, diferenciando os usuários
que teriam um pontecial de impactar o negócio ao deixar uma avaliação.

``` r
plot_ly(yelp_usr_cluster, x = ~year_since, 
               y = ~average_stars,
               z = ~fans, color = ~cluster_usr,
              text = ~paste('Cluster: ', cluster_usr)) %>% 
  add_markers() %>% 
  layout(scene = list(xaxis = list(title = 'No Yelp desde'),
                                   yaxis = list(title = 'Nota Média'),
                                   zaxis = list(title = 'Quantidade de fãs')))
```

<!--html_preserve-->

<div id="htmlwidget-471906a988fdd2a34d50" class="plotly html-widget" style="width:960px;height:960px;">

</div>


<!--/html_preserve-->

A base é então transferida para csv para incluir o número do cluster dos
usuários na base final da modelagem.

``` r
yelp_usr_cluster %>% 
          select(user_id, cluster_usr) %>%
          write.csv(file = "output/usr_cluster.csv")
```

Como próximos passos, seria interessante entender melhor as
características de cada cluster. No relatório em Python é feita uma
análise dos textos das avaliações para cada cluster.

Para classificar usuários que não estão na base selecionada, foi feita
inicialmente uma árvore de classificação.

### 2.4.2. Modelo para definição do cluster do usuário

``` r
user_cluster_tree <- yelp_usr_cluster %>% 
                    select(-user_id) %>% 
                    rpart(cluster_usr ~ ., data = .)

plot_arvore <- as.party(user_cluster_tree)

#plot(plot_arvore)
```

# 3\. Modelagem

## 3.1 Leitura da base final

Após consolidação da base em Python, é feita a leitura da base e
substituição da variável resposta para binária: - Notas maiores ou
iguais a 4 - boas (1) - Notas menores do que 4 - ruim (0)

Além disso, são mantidas apenas as variáveis numéricas para treinamento
da rede neural.

``` r
yelp_raw <- list.files(path = 'output/yelp.csv/', 
                       pattern = "*.csv",
                       full.names = TRUE) %>% 
            map_df(~read_csv(.))

glimpse(yelp_raw)
```

    ## Rows: 219,462
    ## Columns: 63
    ## $ user_id                    <chr> "FLfEG23KQtGOKsv5_4CU9Q", "hjkblI0fn4wmtf2…
    ## $ average_stars              <dbl> 1.00, 3.21, 1.50, 3.99, 4.23, 3.64, 3.67, …
    ## $ compliment_cool            <dbl> 0, 0, 0, 4, 2, 1, 0, 3, 0, 0, 0, 0, 1, 0, …
    ## $ compliment_cute            <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ compliment_funny           <dbl> 0, 0, 0, 4, 2, 1, 0, 3, 0, 0, 0, 0, 1, 0, …
    ## $ compliment_hot             <dbl> 0, 0, 0, 2, 0, 0, 0, 5, 0, 0, 0, 0, 0, 0, …
    ## $ compliment_list            <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ compliment_more            <dbl> 0, 0, 0, 1, 0, 0, 0, 3, 0, 0, 0, 0, 0, 0, …
    ## $ compliment_note            <dbl> 0, 0, 0, 3, 3, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ compliment_photos          <dbl> 0, 0, 0, 1, 3, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ compliment_plain           <dbl> 0, 0, 0, 1, 2, 0, 0, 1, 0, 0, 1, 0, 1, 0, …
    ## $ compliment_profile         <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ compliment_writer          <dbl> 0, 0, 0, 1, 0, 0, 0, 2, 0, 0, 0, 0, 1, 0, …
    ## $ cool                       <dbl> 0, 2, 0, 35, 45, 1, 0, 37, 0, 0, 17, 0, 8,…
    ## $ elite_count                <dbl> 1, 1, 1, 2, 1, 1, 1, 2, 1, 1, 1, 1, 1, 1, …
    ## $ fans                       <dbl> 0, 0, 0, 1, 7, 0, 0, 9, 0, 0, 4, 0, 1, 0, …
    ## $ friends_count              <dbl> 12, 1, 1, 44, 31, 8, 1, 147, 1, 20, 34, 1,…
    ## $ funny                      <dbl> 0, 1, 0, 18, 27, 0, 1, 20, 0, 0, 2, 0, 9, …
    ## $ review_count_usr           <dbl> 1, 19, 2, 105, 279, 14, 3, 190, 2, 4, 38, …
    ## $ useful                     <dbl> 0, 1, 1, 90, 99, 1, 1, 100, 1, 1, 19, 1, 5…
    ## $ year_since                 <dbl> 2017, 2016, 2017, 2011, 2017, 2017, 2015, …
    ## $ tips_counter               <dbl> 0, 0, 0, 1, 1, 5, 1, 9, 0, 1, 0, 0, 5, 0, …
    ## $ total_compliments          <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ cluster_usr                <dbl> 7, 3, 7, 10, 9, 9, 3, 4, 3, 7, 9, 7, 4, 3,…
    ## $ business_id                <chr> "2vBo1wWJckBnGOHhxt9ecg", "2vBo1wWJckBnGOH…
    ## $ stars_rv                   <dbl> 1, 2, 1, 3, 5, 4, 2, 3, 1, 2, 5, 1, 3, 3, …
    ## $ year_rv                    <dbl> 2017, 2018, 2017, 2018, 2019, 2017, 2017, …
    ## $ categories                 <chr> "Sports Bars, Nightlife, Fast Food, Bars, …
    ## $ latitude                   <dbl> 43.64041, 43.64041, 43.64041, 43.64041, 43…
    ## $ longitude                  <dbl> -79.39058, -79.39058, -79.39058, -79.39058…
    ## $ name                       <chr> "St. Louis Bar & Grill", "St. Louis Bar & …
    ## $ review_count               <dbl> 81, 81, 81, 81, 81, 81, 81, 81, 81, 81, 81…
    ## $ stars                      <dbl> 2.5, 2.5, 2.5, 2.5, 2.5, 2.5, 2.5, 2.5, 2.…
    ## $ AcceptsInsurance           <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ AgesAllowed                <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ Alcohol                    <dbl> 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, …
    ## $ BYOB                       <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ BikeParking                <dbl> 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, …
    ## $ BusinessAcceptsCreditCards <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ ByAppointmentOnly          <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ Caters                     <dbl> 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, …
    ## $ CoatCheck                  <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ Corkage                    <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ DogsAllowed                <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ DriveThru                  <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ GoodForDancing             <dbl> 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, …
    ## $ GoodForKids                <dbl> 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, …
    ## $ HappyHour                  <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ HasTV                      <dbl> 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, …
    ## $ NoiseLevel                 <dbl> 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, …
    ## $ OutdoorSeating             <dbl> 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, …
    ## $ RestaurantsAttire          <dbl> 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, …
    ## $ RestaurantsDelivery        <dbl> 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, …
    ## $ RestaurantsGoodForGroups   <dbl> 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, …
    ## $ RestaurantsPriceRange2     <dbl> 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, …
    ## $ RestaurantsReservations    <dbl> 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, …
    ## $ RestaurantsTableService    <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ RestaurantsTakeOut         <dbl> 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, …
    ## $ Smoking                    <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ WheelchairAccessible       <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ WiFi                       <dbl> 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, …
    ## $ tips_counter_bz            <dbl> 14, 14, 14, 14, 14, 14, 14, 14, 14, 14, 14…
    ## $ total_compliments_bz       <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …

``` r
yelp_rv <- yelp_raw %>% 
  #mutate(line = row_number()) %>% 
  select(-'year_rv') %>% 
  mutate(stars_rv = replace(stars_rv >= 4,1,0)) %>% 
  select_if(is.numeric) #%>% sample_frac(0.50)

glimpse(yelp_rv)
```

    ## Rows: 219,462
    ## Columns: 58
    ## $ average_stars              <dbl> 1.00, 3.21, 1.50, 3.99, 4.23, 3.64, 3.67, …
    ## $ compliment_cool            <dbl> 0, 0, 0, 4, 2, 1, 0, 3, 0, 0, 0, 0, 1, 0, …
    ## $ compliment_cute            <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ compliment_funny           <dbl> 0, 0, 0, 4, 2, 1, 0, 3, 0, 0, 0, 0, 1, 0, …
    ## $ compliment_hot             <dbl> 0, 0, 0, 2, 0, 0, 0, 5, 0, 0, 0, 0, 0, 0, …
    ## $ compliment_list            <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ compliment_more            <dbl> 0, 0, 0, 1, 0, 0, 0, 3, 0, 0, 0, 0, 0, 0, …
    ## $ compliment_note            <dbl> 0, 0, 0, 3, 3, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ compliment_photos          <dbl> 0, 0, 0, 1, 3, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ compliment_plain           <dbl> 0, 0, 0, 1, 2, 0, 0, 1, 0, 0, 1, 0, 1, 0, …
    ## $ compliment_profile         <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ compliment_writer          <dbl> 0, 0, 0, 1, 0, 0, 0, 2, 0, 0, 0, 0, 1, 0, …
    ## $ cool                       <dbl> 0, 2, 0, 35, 45, 1, 0, 37, 0, 0, 17, 0, 8,…
    ## $ elite_count                <dbl> 1, 1, 1, 2, 1, 1, 1, 2, 1, 1, 1, 1, 1, 1, …
    ## $ fans                       <dbl> 0, 0, 0, 1, 7, 0, 0, 9, 0, 0, 4, 0, 1, 0, …
    ## $ friends_count              <dbl> 12, 1, 1, 44, 31, 8, 1, 147, 1, 20, 34, 1,…
    ## $ funny                      <dbl> 0, 1, 0, 18, 27, 0, 1, 20, 0, 0, 2, 0, 9, …
    ## $ review_count_usr           <dbl> 1, 19, 2, 105, 279, 14, 3, 190, 2, 4, 38, …
    ## $ useful                     <dbl> 0, 1, 1, 90, 99, 1, 1, 100, 1, 1, 19, 1, 5…
    ## $ year_since                 <dbl> 2017, 2016, 2017, 2011, 2017, 2017, 2015, …
    ## $ tips_counter               <dbl> 0, 0, 0, 1, 1, 5, 1, 9, 0, 1, 0, 0, 5, 0, …
    ## $ total_compliments          <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ cluster_usr                <dbl> 7, 3, 7, 10, 9, 9, 3, 4, 3, 7, 9, 7, 4, 3,…
    ## $ stars_rv                   <dbl> 0, 0, 0, 0, 1, 1, 0, 0, 0, 0, 1, 0, 0, 0, …
    ## $ latitude                   <dbl> 43.64041, 43.64041, 43.64041, 43.64041, 43…
    ## $ longitude                  <dbl> -79.39058, -79.39058, -79.39058, -79.39058…
    ## $ review_count               <dbl> 81, 81, 81, 81, 81, 81, 81, 81, 81, 81, 81…
    ## $ stars                      <dbl> 2.5, 2.5, 2.5, 2.5, 2.5, 2.5, 2.5, 2.5, 2.…
    ## $ AcceptsInsurance           <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ AgesAllowed                <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ Alcohol                    <dbl> 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, …
    ## $ BYOB                       <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ BikeParking                <dbl> 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, …
    ## $ BusinessAcceptsCreditCards <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ ByAppointmentOnly          <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ Caters                     <dbl> 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, …
    ## $ CoatCheck                  <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ Corkage                    <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ DogsAllowed                <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ DriveThru                  <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ GoodForDancing             <dbl> 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, …
    ## $ GoodForKids                <dbl> 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, …
    ## $ HappyHour                  <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ HasTV                      <dbl> 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, …
    ## $ NoiseLevel                 <dbl> 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, 4, …
    ## $ OutdoorSeating             <dbl> 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, …
    ## $ RestaurantsAttire          <dbl> 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, …
    ## $ RestaurantsDelivery        <dbl> 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, …
    ## $ RestaurantsGoodForGroups   <dbl> 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, …
    ## $ RestaurantsPriceRange2     <dbl> 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, …
    ## $ RestaurantsReservations    <dbl> 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, …
    ## $ RestaurantsTableService    <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ RestaurantsTakeOut         <dbl> 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, 2, …
    ## $ Smoking                    <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ WheelchairAccessible       <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ WiFi                       <dbl> 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, 3, …
    ## $ tips_counter_bz            <dbl> 14, 14, 14, 14, 14, 14, 14, 14, 14, 14, 14…
    ## $ total_compliments_bz       <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, …

## 3.2 Bases de Treino e Teste

A base é divida em treino, validação e teste com a função `split` do
pacote tidymodels.

``` r
split <- initial_split(yelp_rv, prop = 0.8 , strata = stars_rv)

train_val <- training(split)


split_val <- initial_split(train_val, prop = 0.5, strata = stars_rv)

yelp_train <- training(split_val)
yelp_val <- testing(split_val)
yelp_test <- testing(split)
```

Calcula-se então a média e desvio padrão da base de treino para utilizar
na normalização. (Chollet)

``` r
mean <- yelp_train %>% 
        select(-stars_rv) %>% 
        apply(., 2, mean) 

std <- yelp_train %>% 
        select(-stars_rv) %>% 
        apply(., 2, sd)
```

Por fim, as bases são normalizadas e transformadas em matriz para
otimizar os cálculos na rede neural

``` r
x_train <- yelp_train %>% 
            select(-stars_rv) %>% 
            scale(center = mean, scale = std) %>% 
            as.matrix()

dim(x_train)
```

    ## [1] 87786    57

``` r
y_train <- yelp_train %>% 
            select(stars_rv) %>% 
            as.matrix()

x_val <-  yelp_val %>% 
            select(-stars_rv) %>% 
            scale(center = mean, scale = std) %>% 
            as.matrix()

dim(x_val)
```

    ## [1] 87785    57

``` r
y_val <- yelp_val %>% 
            select(stars_rv) %>% 
            data.matrix()

dim(x_val)
```

    ## [1] 87785    57

``` r
x_test <- yelp_test %>% 
          select(-stars_rv) %>% 
          scale(center = mean, scale = std) %>% 
          as.matrix()

dim(x_test) 
```

    ## [1] 43891    57

``` r
y_test <- yelp_test %>% 
            select(stars_rv) %>% 
            data.matrix()
```

## 3.3. Rede Neural

Foram testadas diferentes estruturas de rede neural. De qualquer forma,
a última camada possui uma função de ativação `sigmoid` para que a
resposta seja uma probabilidade de acerto.

``` r
rm(yelp_nn)

yelp_nn <- keras_model_sequential() %>% 
  layer_dense(units = 30, activation = "tanh", input_shape = ncol(x_train)) %>%
  layer_dropout(rate = 0.5) %>%
  layer_dense(units = 16, activation = "relu") %>%
  #layer_dropout(rate = 0.5) %>%
  layer_dense(units = 16, activation = "relu") %>%
  #layer_dense(units = 6, activation = "softmax")
  layer_dense(units = 1, activation = "sigmoid")

yelp_nn %>% 
  compile(optimizer = "rmsprop", 
          #loss = "sparse_categorical_crossentropy", 
          loss = "binary_crossentropy",
          metrics = c("accuracy"))


history <- yelp_nn %>% 
  fit(x_train, y_train, 
      epochs = 40, batch_size = 512, 
      validation_data = list(x_val, y_val))

plot(history)
```

<img src="Modelagem-Yelp-Final_files/figure-gfm/unnamed-chunk-20-1.png" width="960" />

``` r
#keras::get_weights(yelp_nn)

(results <- yelp_nn %>% evaluate(x_test, y_test))
```

    ##      loss  accuracy 
    ## 0.4625269 0.7709326

Foi adicionada uma camada de dropout para diminuir o overfit do modelo.
Observa-se que foi efeciente, pois a perda da base de validação não
ultrappassa a perda da base de treino. Ainda assim, mais testes de
estruturas e otimização poderiam ser feitos para melhorar a performance
do modelo.

## 3.4. Desempenho do modelo

Com o modelo pronto, é feito um teste na respectiva base. Sua
performance é calculada com a área sob a curva ROC.

``` r
resultados <- tibble(observado = factor(y_test)) %>% 
  bind_cols(data.frame(prob = predict(yelp_nn, as.matrix(x_test))))

resultados %>% 
  roc_auc(observado, prob)
```

    ## # A tibble: 1 x 3
    ##   .metric .estimator .estimate
    ##   <chr>   <chr>          <dbl>
    ## 1 roc_auc binary         0.837

``` r
resultados %>% 
  roc_curve(observado, prob) %>% 
  autoplot()
```

<img src="Modelagem-Yelp-Final_files/figure-gfm/unnamed-chunk-21-1.png" width="960" />

Pelo gráfico, observa-se que o modelo atingiu um desempenho satisfatório
na base de teste, considerando que é um problema de recomendação. A
seguir, a matriz de confusão do modelo, considerando o corte de 0.50,
pois é preferível minimizar os falsos negativos (recomendar um
estabelecimento que o usuário não irá gostar) do que os falsos positivos
(deixar de recomendar um restaurante que o usuário irá gostar, que pode
ser um que ele já tenha avaliado).

``` r
resultados %>% 
  mutate(.pred = if_else(prob >= 0.5,1,0)) %>% 
  mutate(.pred = as.factor(.pred)) %>% 
  conf_mat(truth = observado, estimate = .pred) %>% 
  autoplot()
```

<img src="Modelagem-Yelp-Final_files/figure-gfm/unnamed-chunk-22-1.png" width="960" />

``` r
?case
```

# 4\. Recomendação

## 4.1. Usuário criado

Abaixo é criado um usuário com as mesmas informações existentes na base
de usuários de forma aleatória. Para definir seu cluster, é utilizada a
árvore criada anteriormente.

``` r
glimpse(yelp_usr_cluster)
```

    ## Rows: 119,792
    ## Columns: 24
    ## $ user_id            <chr> "-4Anvj46CWf57KWI9UQDLg", "-BUamlG3H-7yqpAl1p-msw"…
    ## $ average_stars      <dbl> 3.50, 1.50, 3.00, 3.56, 3.00, 4.00, 4.17, 3.57, 4.…
    ## $ compliment_cool    <dbl> 0, 0, 0, 0, 0, 0, 0, 169, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ compliment_cute    <dbl> 0, 0, 0, 0, 0, 0, 0, 2, 0, 0, 0, 0, 0, 0, 0, 0, 0,…
    ## $ compliment_funny   <dbl> 0, 0, 0, 0, 0, 0, 0, 169, 0, 0, 0, 0, 0, 0, 0, 0, …
    ## $ compliment_hot     <dbl> 0, 0, 0, 0, 0, 0, 0, 94, 0, 0, 0, 0, 0, 0, 0, 2, 0…
    ## $ compliment_list    <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,…
    ## $ compliment_more    <dbl> 0, 0, 0, 0, 0, 0, 0, 3, 0, 0, 0, 0, 0, 0, 0, 0, 0,…
    ## $ compliment_note    <dbl> 0, 0, 1, 0, 0, 0, 0, 16, 0, 1, 0, 0, 0, 0, 0, 1, 0…
    ## $ compliment_photos  <dbl> 0, 0, 0, 0, 0, 0, 0, 97, 0, 0, 0, 0, 0, 0, 0, 0, 0…
    ## $ compliment_plain   <dbl> 0, 0, 0, 0, 0, 0, 0, 66, 0, 0, 0, 1, 0, 0, 0, 0, 0…
    ## $ compliment_profile <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,…
    ## $ compliment_writer  <dbl> 0, 0, 0, 0, 0, 0, 0, 30, 0, 0, 0, 0, 0, 0, 0, 0, 0…
    ## $ cool               <dbl> 2, 0, 1, 0, 1, 0, 0, 1562, 2, 1, 1, 9, 0, 5, 0, 9,…
    ## $ elite_count        <dbl> 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1, 1,…
    ## $ fans               <dbl> 1, 0, 0, 0, 0, 0, 0, 39, 0, 0, 0, 1, 0, 0, 0, 0, 0…
    ## $ friends_count      <dbl> 1, 16, 15, 27, 1, 1, 1, 338, 59, 6, 10, 100, 8, 1,…
    ## $ funny              <dbl> 0, 0, 1, 0, 0, 0, 0, 1266, 3, 1, 4, 0, 1, 1, 1, 5,…
    ## $ review_count_usr   <dbl> 2, 2, 4, 27, 2, 6, 6, 66, 28, 3, 8, 37, 4, 20, 1, …
    ## $ useful             <dbl> 2, 0, 1, 5, 1, 3, 16, 1683, 12, 1, 2, 30, 4, 30, 0…
    ## $ year_since         <dbl> 2016, 2016, 2011, 2019, 2014, 2017, 2014, 2019, 20…
    ## $ tips_counter       <dbl> 0, 1, 0, 0, 0, 1, 0, 0, 0, 19, 0, 0, 0, 0, 0, 2, 0…
    ## $ total_compliments  <dbl> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0,…
    ## $ cluster_usr        <int> 3, 7, 10, 9, 3, 9, 4, 9, 4, 10, 3, 10, 4, 3, 7, 9,…

``` r
rm(user)

compliment_max <- 100

user <- tibble(user_id = 'random_user',
               average_stars = round(runif(1, 1.0, 5),2),
               compliment_cool = ceiling(runif(1,0, compliment_max)),
               compliment_cute = ceiling(runif(1,0, compliment_max)),
               compliment_funny = ceiling(runif(1,0, compliment_max)),
               compliment_hot  = ceiling(runif(1,0, compliment_max)),
               compliment_list = ceiling(runif(1,0, compliment_max)),
               compliment_more = ceiling(runif(1,0, compliment_max)),
               compliment_note = ceiling(runif(1,0, compliment_max)),
               compliment_photos = ceiling(runif(1,0, compliment_max)),
               compliment_plain = ceiling(runif(1,0, compliment_max)),
               compliment_profile = ceiling(runif(1,0, compliment_max)),
               compliment_writer = ceiling(runif(1,0, compliment_max)),
               cool = ceiling(runif(1,0, compliment_max)),
               elite_count = 0,
               fans = ceiling(runif(1,0, compliment_max)),
               friends_count = ceiling(runif(1,0, compliment_max)),
               funny = ceiling(runif(1,0, compliment_max)),
               review_count_usr = ceiling(runif(1,0,compliment_max)),
               useful = ceiling(runif(1,0, compliment_max)),
               year_since = ceiling(runif(1,2004, 2019)),
               tips_counter = ceiling(runif(1,0, compliment_max)),
               total_compliments = ceiling(runif(1,0, compliment_max))
                )

## criação aleatória do número de anos que o usuário foi elite
user$elite_count <- ceiling(runif(1,0, (2020-user$year_since)))

#encontra o número do cluster em que o usuário se encaixa
user$cluster_usr <- user_cluster_tree %>%
      predict(user) %>% 
      ceiling()
```

Usuário criado:

``` r
glimpse(user)
```

    ## Rows: 1
    ## Columns: 24
    ## $ user_id            <chr> "random_user"
    ## $ average_stars      <dbl> 4.07
    ## $ compliment_cool    <dbl> 27
    ## $ compliment_cute    <dbl> 34
    ## $ compliment_funny   <dbl> 3
    ## $ compliment_hot     <dbl> 27
    ## $ compliment_list    <dbl> 26
    ## $ compliment_more    <dbl> 38
    ## $ compliment_note    <dbl> 74
    ## $ compliment_photos  <dbl> 97
    ## $ compliment_plain   <dbl> 92
    ## $ compliment_profile <dbl> 61
    ## $ compliment_writer  <dbl> 47
    ## $ cool               <dbl> 1
    ## $ elite_count        <dbl> 7
    ## $ fans               <dbl> 42
    ## $ friends_count      <dbl> 49
    ## $ funny              <dbl> 14
    ## $ review_count_usr   <dbl> 31
    ## $ useful             <dbl> 64
    ## $ year_since         <dbl> 2010
    ## $ tips_counter       <dbl> 30
    ## $ total_compliments  <dbl> 48
    ## $ cluster_usr        <dbl> 8

Em seguida, a partir do número de reviews gerado, é selecionada
aleatóriamente a mesma quantidadade de estabelecimentos da base
business e atribuídas notas de review aleatórias. Depois, as informações
do usuário e dos estabelcimentos são consolidadas em uma base única,
como se fosse uma amostra da base
original.

``` r
# seleção aleatória de estabelecimentos e notas atribuídas a cada um baseado no número de reviews

n_reviews <- user$review_count_usr

reviewed_usr <- tibble(business_id = sample(yelp_bz_raw$business_id, 
                                            n_reviews), #seleçao aleatória de estabelecimentos
                                            stars_rv = ceiling(runif(n_reviews, 1.0, 5)), #nota
                                            year_rv = ceiling(runif(n_reviews, 2009, 2019)), #ano da review
                                            )
user_hist <- user %>% 
            bind_rows(replicate(n_reviews-1, user, simplify = FALSE)) %>% #replica as informações do usuário
            bind_cols(reviewed_usr) %>% #junta os estabelecimentos e notas dadas
            left_join(., yelp_bz_raw, by = 'business_id') #junta as informações dos estabelecimentos

#glimpse(user_hist)
```

## 4.2. Função para recomendação

Para gerar as recomendações, foi criada uma função para selecionar os
possíveis restaurantes que o usuário gostaria de ir e calcular a
probabilidade de uma boa avaliação. Como dados de entrada, devem ser
fornecidas as informações do perfil do usuário no yelp e as avaliações
de estabelecimentos já visitados.

A partir dessas informações, é gerada a base `to_go` que, a partir do
cluster do usuário, filtra os estabelecimentos que ele poderia ir de
acordo com as boas avaliações feitas por usuários de perfil semelehante
(do mesmo cluster)

Em seguida, na base `to_review`, são cruzadas as informações de
avaliações do usuário com a base anterior para criar uma matriz
contendo as informações do usuário e dos estabelecimentos que ele ainda
não avaliou. Essa base é então normalizada e transoformada em matriz
para input no modelo pela base `user_x_test`.

As previsões de probabilidade do usuário dar uma nota maior ou igual a 4
para os estabelcimentos que ele ainda não visitou são então armazenadas
no vetor `predictions`, que é então adicionado à base `to_review`,
criando uma tabela única que será utilizada como fonte para as
recommendações. Por fim, são filtrados os estabelecimentos com
probabilidade maior que 50%.

``` r
recomm_f <- function(user, reviewed_usr){
  
  to_go <- yelp_raw %>% 
    filter(stars_rv >= 4) %>% 
    filter(cluster_usr == user$cluster_usr) %>% 
    select(business_id) %>% 
    distinct() 
  
  n_go <- nrow(to_go)
  
  #filtra todos os estabelecimentos do cluster do usuário e junta as informações para modelagem
  to_review <- user %>% 
            bind_rows(replicate(n_go-1, user, simplify = FALSE)) %>% #replica as informações do usuário
            bind_cols(to_go) %>% #junta os estabelecimentos e notas dadas
            left_join(., yelp_bz_raw, by = 'business_id')
  
  #prepara a base para o modelo
  user_x_test <- to_review %>% 
          select_if(is.numeric) %>% 
          #select(-stars_rv) %>% 
          scale(center = mean, scale = std) %>% 
          as.matrix()

  #aplica a base no modelo
  predictions <- as_tibble(predict(yelp_nn, user_x_test))
  
  
  #seleciona as principais recomendações
  recommendation <- to_review %>% 
    bind_cols(pred = predictions) %>% 
    anti_join(., reviewed_usr, by = 'business_id') %>% 
    filter(V1 > 0.5)

}
```

### 4.2.1. Recomendação para usuário criado

A função é então aplicada ao usuário criado com informações aleatórias.
Os 5 estabelecimentos com maior probabilidade de avaliação positiva e
sua localização são os seguintes.

``` r
rec_new <- recomm_f(user,reviewed_usr)
```

``` r
rec_new %>% 
  arrange(-V1) %>% 
  select(name, categories, V1)

Í#mapa com as top 5 recomendações

top_5 <- rec_new %>% 
    top_n(5, V1) %>%
    arrange(-V1) %>% 
    mutate(rank = as.factor(row_number()))

top_5 %>% 
  select(name, categories, V1)

qmplot(longitude, latitude, data = top_5, 
       maptype = "toner-background", 
       color = rank,
       size = V1)
```

## 4.3. Usuário aleatório da base

Para validar as recomendações, é feito o teste também com um usuário
aleatório da base de teste.

``` r
n <- ceiling(runif(1,1,nrow(yelp_test)))


(random_user <- yelp_raw[n,]$user_id)
```

    ## [1] "98EWykvvVTqvO_o8J0cGLA"

``` r
user2 <- yelp_usr_cluster %>% 
            filter(user_id == random_user)

glimpse(user2)
```

    ## Rows: 1
    ## Columns: 24
    ## $ user_id            <chr> "98EWykvvVTqvO_o8J0cGLA"
    ## $ average_stars      <dbl> 3.6
    ## $ compliment_cool    <dbl> 0
    ## $ compliment_cute    <dbl> 0
    ## $ compliment_funny   <dbl> 0
    ## $ compliment_hot     <dbl> 0
    ## $ compliment_list    <dbl> 0
    ## $ compliment_more    <dbl> 0
    ## $ compliment_note    <dbl> 0
    ## $ compliment_photos  <dbl> 0
    ## $ compliment_plain   <dbl> 0
    ## $ compliment_profile <dbl> 0
    ## $ compliment_writer  <dbl> 0
    ## $ cool               <dbl> 0
    ## $ elite_count        <dbl> 1
    ## $ fans               <dbl> 0
    ## $ friends_count      <dbl> 1
    ## $ funny              <dbl> 2
    ## $ review_count_usr   <dbl> 4
    ## $ useful             <dbl> 5
    ## $ year_since         <dbl> 2018
    ## $ tips_counter       <dbl> 0
    ## $ total_compliments  <dbl> 0
    ## $ cluster_usr        <int> 9

``` r
reviewed_usr2 <- yelp_raw %>% 
  filter(user_id == random_user)

glimpse(reviewed_usr)
```

    ## Rows: 31
    ## Columns: 3
    ## $ business_id <chr> "r-kj-kBSKFKh0sM8EVX8AA", "H7rpWv02D6WTu6IpNNDkWw", "9YnM…
    ## $ stars_rv    <dbl> 5, 5, 3, 2, 2, 5, 4, 3, 4, 2, 5, 4, 2, 3, 2, 5, 5, 5, 5, …
    ## $ year_rv     <dbl> 2012, 2015, 2012, 2012, 2018, 2010, 2014, 2015, 2014, 201…

``` r
rec_user <- recomm_f(user2,reviewed_usr2)
```

### 4.3.1. Recomendação para usuário da base

``` r
#top 5 recomendações

  rec_user %>% 
    top_n(5, V1) %>%
    arrange(V1) %>% 
    mutate(rank = as.factor(row_number())) %>% 
    ggplot(aes(x = V1, y = name)) +
    geom_col() +
    labs(x = "Probabilidade de avaliação positiva",
    y = 'Recomendação')
```

<img src="Modelagem-Yelp-Final_files/figure-gfm/unnamed-chunk-30-1.png" width="960" />

``` r
top_5 <- rec_user %>% 
    top_n(5, V1) %>%
    arrange(-V1) %>% 
    mutate(rank = as.factor(row_number()))

top_5 %>% 
  select(name, categories, V1)
```

    ## # A tibble: 5 x 3
    ##   name                        categories                                      V1
    ##   <chr>                       <chr>                                        <dbl>
    ## 1 Duotherapy                  Physical Therapy, Health & Medical, Massage… 0.968
    ## 2 Downsview Chiropractic      Doctors, Massage Therapy, Naturopathic/Holi… 0.968
    ## 3 NewDermaMed Cosmetic and A… Laser Hair Removal, Medical Spas, Beauty & … 0.968
    ## 4 Daniel Chizick Massage The… Massage Therapy, Beauty & Spas, Health & Me… 0.967
    ## 5 Nancy Bishay, DDS           Orthodontists, Dentists, Oral Surgeons, Gen… 0.966

``` r
qmplot(longitude, latitude, data = top_5, 
       maptype = "toner-background", 
       color = rank,
       size = V1)
```

<img src="Modelagem-Yelp-Final_files/figure-gfm/unnamed-chunk-31-1.png" width="960" />

### 4.3.2.Recomendação por categoria

O ideal seria fornecer recomendações de acordo com o o que o usuário
procura. Por isso, é mostrado abaixo os top 5 estabelecimentos de
diferentes categorias.

``` r
rec_user %>% 
  select(name, categories, V1) %>% 
  unnest_tokens(category, categories) %>% 
  filter(category %in% c('food','restaurants','bars','pub')) %>% 
  group_by(category) %>%
  mutate(pred_avg = mean(V1)) %>% 
  arrange(desc(V1)) %>% 
  unique() %>% 
  slice(1:5) %>%
  ggplot(aes(V1, name, fill = category)) +
  geom_col(show.legend = FALSE) +
  facet_wrap(~category, scales = 'free') +
  scale_x_continuous() +
  scale_y_reordered() +
  labs(x = 'Probabilidade de boa avaliação')
```

<img src="Modelagem-Yelp-Final_files/figure-gfm/unnamed-chunk-32-1.png" width="960" />

# 5\. Conclusão

Foram avaliadas todas as bases do dataset, mas nem todas as informações
disponíveis foram utilizadas no modelo de redes neurais do sistema de
recomendação. Após entender com uma análise de componentes principais o
impacto dos atributos dos estabelecimentos na nota, foi feita uma
clusterização dos usuários para identificar os perfis semelhantes e
utilizar os locais frequentados como base para as recomendações. Dessa
forma, foi elaborado um classificador para indicação de estabelecimentos
por diferentes categorias, de acordo com o que o usuário desejasse
visitar.

## 5.1 Oportunidades de melhorias:

1.  Utilizar as informações de movimento dos estabelecimentos e cruzá-la
    com os horários em que as avaliações foram feitas.
2.  Incluir texto de reviews e tips no modelo pela presença de palavras
    palavras-chave, por modelos de tópicos, por uma análise de
    sentimentos, ou por “encoding” do texto completo.
3.  Criar interface de usuários com Shiny (R).
4.  Replicar o algoritmo para outras cidades e estabelecendo
5.  Utilizar a localização do usuário na busca por recomendação.

# 6\. Referências

  - Neumann, D. Material de aula do cursos Big Data e Computação em
    Nuvem

  - Mendonça, T. Material de aula do curso Modelagem Preditiva Avançada

  - [Documentação
    PySpark](http://spark.apache.org/docs/latest/api/python/pyspark.sql.html)

  - [StackOverflow](https://stackoverflow.com) - base colaborativa de
    programadores para solução de erros

  - [Fernandez, P. Marques. P. Data Science, Marketing and
    Business](https://datascience.insper.edu.br/datascience.pdf)

  - [Rahimi, S.; Mottahedi, S.; Liu, X. The Geography of Taste: Using
    Yelp to Study Urban Culture. ISPRS Int. J.
    Geo-Inf. 2018, 7, 376.](https://www.mdpi.com/2220-9964/7/9/376?type=check_update&version=1)

  - [Chollet, F. et al, Deep Learning with
    R](https://www.manning.com/books/deep-learning-with-r)

  - [Silge, J., Topic modeling of Sherlock Holmes
    stories](https://juliasilge.com/blog/sherlock-holmes-stm/)

  - <https://www.datanovia.com/en/lessons/clustering-distance-measures/>

  - [K-means clustering with tidy data
    principles](https://www.tidymodels.org/learn/statistics/k-means/)

  - Arquivos disponíveis no
    [repositório](https://github.com/sanchezvivi/pads-yelp)