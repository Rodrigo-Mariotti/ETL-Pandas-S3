# Análise de vítimas de acidentes de trânsito com AWS

Projeto de ETL e análise de dados públicos de acidentes de trânsito, desenvolvido em notebooks Python com Pandas e Boto3. O processo filtra e trata os registros de feriados nacionais, envia o CSV tratado para um bucket Amazon S3 e realiza consultas sobre óbitos, datas e estados.

---

## Relatório

Projeto desenvolvido durante um Programa de estágio usando Notebook Python e a biblioteca Boto3 para iteração com Buckets S3 da AWS

## Entregáveis
[x] [etapa1.ipynb](tratamento.ipynb)  
[x] [etapa2.ipynb](analise.ipynb)  
[x] [csv bruto 0,01%](Vitimas_resumidas.csv)  
[x] [csv limpo](Vitimas_tratado.csv)  


## Etapas

**0. Sobre o dados escolhidos:**  
O arquivo csv original contém 17 colunas e 10 milhões de linhas, tendo o tamanho de 1.33GB. Os dados são referentes às vítimas de acidentes de trânsito durante o período de 2018 até 2024, este repositório contém uma cópia com apenas as primeiras 1000 linhas do csv original.


**1. Reduzir csv e upload AWS:**  
Para reduzir o processamento usado pela AWS decidi retirar as colunas que não usarei, também vou reduzir as linhas, mantendo apenas aquelas cuja data seja condizente com os principais feriados nacionais:  
Confraternização Universal (1º de janeiro)  
Tiradentes (21 de abril)  
Dia do Trabalhador (1º de maio)  
Independência do Brasil (7 de setembro)  
Nossa Senhora Aparecida (12 de outubro)  
Finados (2 de novembro)  
Natal (25 de dezembro)  

Colunas usadas:

0 ~~num_acidente;~~  
1 ~~chv_localidade;~~  
2 data_acidente;  
3 uf_acidente;  
4 ~~ano_acidente;~~  
5 ~~mes_acidente;~~  
6 ~~mes_ano_acidente;~~  
7 faixa_idade;  
8 genero;	  
9 ~~tp_envolvido;~~  
10 ~~gravidade_lesao;~~  
11 ~~equip_seguranca;~~  
12 ~~ind_motorista;~~  
13 ~~susp_alcool;~~  
14 qtde_envolvidos;  
15 ~~qtde_feridosilesos;~~  
16 qtde_obitos

Usei localmente este código para criar um csv menor, tendo apenas 5 colunas e pouco menos de 150 mil linhas, também abreviei dados repetitivos.

```py
import pandas as pd

df = pd.read_csv("Vitimas_DadosAbertos_20250112.csv", sep=";", usecols=[2, 3, 7, 8, 16])

# Tratamento data
df["data_acidente"] = pd.to_datetime(df["data_acidente"])
feriados = {(1, 1), (4, 21), (5, 1), (9, 7), (10, 12), (11, 2), (12, 25)}
df = df[df["data_acidente"].dt.month.combine(df["data_acidente"].dt.day, lambda m, d: (m, d)).isin(feriados)]

# Tratamento gênero
df["genero"] = df["genero"].replace({
    "MASCULINO": "M",
    "FEMININO": "F",
    r"NAO INFORMADO|DESCONHECIDO": "NA"
}, regex=True)

# Tratamento idade
df["faixa_idade"] = df["faixa_idade"].replace({
    "NAO INFORMADO": "NA",
    r"MAIOR QUE 80": "80+",
    r"ENTRE\s|\sANOS": "",
    r"\sE\s": "-"
}, regex=True)


df.to_csv("Vitimas_tratado.csv", index=False)
```
Arquivo gerado: [csv limpo](dados\Vitimas_tratado.csv)

Para autenticar o acesso local à AWS, instalei o AWS CLI. Segue abaixo o print da configuração pelo terminal:  
![evidência 1](../Evidencias\evidencia1.png)  
Porém ao executar o código abaixo obitive um erro relacionado às credenciais da AWS, após múltiplos erros mudei o método, desta vez configurei usando o comando -aws configure sso-, que resultou em um erro semelhante, porém na segunda tentativa funcionou.  
![evidência 2](../Evidencias\evidencia2.png)  
_Observação: Após a primeira tentativa eu mudei o nome do perfil de desafio4 para localdesafio4._


Depois gerei o bucket e fiz o upload do arquivo tratado usando este código:

```py
import boto3


AWS_PROFILE = "localdesafio4"
BUCKET_NAME = "dados-renavam"
CSV_FILE = "Vitimas_tratado.csv"
OBJETO_S3 = "dados_acidentes.csv"


session = boto3.Session(profile_name=AWS_PROFILE)
s3_client = session.client("s3")

# Criação do bucket
try:
    s3_client.create_bucket(
        Bucket=BUCKET_NAME,
        CreateBucketConfiguration={'LocationConstraint': session.region_name}
    )
    print(f"Bucket '{BUCKET_NAME}' criado com sucesso!")
except s3_client.exceptions.BucketAlreadyOwnedByYou:
    print(f"Bucket '{BUCKET_NAME}' já existe.")

# Upload do arquivo
try:
    s3_client.upload_file(CSV_FILE, BUCKET_NAME, OBJETO_S3)
    print(f"Arquivo '{CSV_FILE}' enviado para '{BUCKET_NAME}/{OBJETO_S3}' com sucesso!")
except Exception as e:
    print("Erro ao enviar arquivo:", e)
```
_Observação: Ambos os códigos estão documentados no notebook da primeira etapa._
Notebook da primeira etapa: [tratamento](tratamento.ipynb)

**2. Definindo questionamentos:**  
Requisitos:
- Uma cláusula que filtra dados usando ao menos dois operadores lógicos;  
- Uma função de agregação; 
- Uma função condicional; 
- Uma função de conversão; 
- Uma função de data;  
- Uma função de string. 


Questinamentos:  
- 1-Vitimas fatais por faixa de idade e gênero
- 2-Acidentes durante feriados ao passar dos anos
- 3-Relação de obitos e acidentes por Estado

Notebook da segunda etapa: [análise](analise.ipynb)  

_Observação: Ao desenvolver a segunda etapa, pensei que não precisaria definir a seção da AWS novamente, pois a mesma estaria armazenda no kernel após a primeira etapa, mas ao executar o notebook, foi provado que minha lógica estava errada. Tive que copiar o trecho do código da primeira etapa para a execução da segunda etapa funcionar._

Output da análise:
```txt
Óbitos por gênero e faixa de idade entre 2018 e 2024:

 faixa_idade  feminino  masculino
       1-17        25         76
      18-24        62        369
      25-29        46        230
      30-34        38        184
      35-39        43        192
      40-44        29        158
      45-49        25        109
      50-54        21        125
      55-59        13         76
      60-64        15         68
      65-69        15         55
      70-74        14         37
      75-79         8         17
        80+        14         28


Acidentes durante feriados ao passar dos anos:

 data_acidente                    feriado  qtde_acidentes
   2018-01-01 Confraternização Universal            2536
   2018-04-21                 Tiradentes            2748
   2018-05-01         Dia do Trabalhador            2159
   2018-09-07    Independência do Brasil            2896
   2018-10-12    Nossa Senhora Aparecida            2781
   2018-11-02                    Finados            2464
   2018-12-25                      Natal            2438
   2019-01-01 Confraternização Universal            3019
   2019-04-21                 Tiradentes            3884
   2019-05-01         Dia do Trabalhador            3116
   2019-09-07    Independência do Brasil            4628
   2019-10-12    Nossa Senhora Aparecida            4702
   2019-11-02                    Finados            4250
   2019-12-25                      Natal            3236
   2020-01-01 Confraternização Universal            3505
   2020-04-21                 Tiradentes            1894
   2020-05-01         Dia do Trabalhador            2673
   2020-09-07    Independência do Brasil            3149
   2020-10-12    Nossa Senhora Aparecida            3394
   2020-11-02                    Finados            3148
   2020-12-25                      Natal            3402
   2021-01-01 Confraternização Universal            3552
   2021-04-21                 Tiradentes            2919
   2021-05-01         Dia do Trabalhador            4403
   2021-09-07    Independência do Brasil            3448
   2021-10-12    Nossa Senhora Aparecida            3491
   2021-11-02                    Finados            3142
   2021-12-25                      Natal            3571
   2022-01-01 Confraternização Universal            3539
   2022-04-21                 Tiradentes            3268
   2022-05-01         Dia do Trabalhador            3945
   2022-09-07    Independência do Brasil            3728
   2022-10-12    Nossa Senhora Aparecida            3423
   2022-11-02                    Finados            2847
   2022-12-25                      Natal            3448
   2023-01-01 Confraternização Universal            3327
   2023-04-21                 Tiradentes            3098
   2023-05-01         Dia do Trabalhador            2833
   2023-09-07    Independência do Brasil            3481
   2023-10-12    Nossa Senhora Aparecida            3183
   2023-11-02                    Finados            2839
   2023-12-25                      Natal            3035
   2024-01-01 Confraternização Universal            3027
   2024-04-21                 Tiradentes            3316
   2024-05-01         Dia do Trabalhador            2922
   2024-09-07    Independência do Brasil            3467


Relação de acidentes por estado:

 estado  total_obitos  total_acidentes  percentual_obitos
    AC             3              548               0.55
    AL            85              391              21.74
    AM            13              223               5.83
    AP             1              239               0.42
    BA           187             2809               6.66
    CE           167             2276               7.34
    DF            45             4979               0.90
    ES            93             6015               1.55
    GO           136            14583               0.93
    MA           136              479              28.39
    MG           301            40568               0.74
    MS            26             2746               0.95
    MT            96              660              14.55
    PA           196             3058               6.41
    PB           132              456              28.95
    PE           218             1780              12.25
    PI            60             1526               3.93
    PR           187            16404               1.14
    RJ           206             2293               8.98
    RN            65              622              10.45
    RO            61             2700               2.26
    RR            15              603               2.49
    RS           200              378              52.91
    SC           115            15807               0.73
    SE            12              157               7.64
    SP           901            25924               3.48
    TO            38             1050               3.62
```
