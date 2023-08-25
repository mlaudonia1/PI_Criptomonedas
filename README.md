## hacer una dataframe con la serie de tiempo de las 10 monedas, y base de dats categorias, qeudarse solo con capitalziacion de mercado (quitar 39),  y columnas nombre, contenido, cap mercado, volumen

# %%
# URL de la API de CoinGecko para obtener datos de criptomonedas
api_url = 'https://api.coingecko.com/api/v3/coins/markets'

# Parámetros de la solicitud
params = {
    'vs_currency': 'usd',  # Moneda base (USD)
    'order': 'market_cap_desc',  # Ordenar por capitalización de mercado descendente
    'per_page': 10,  # Número de resultados por página
    'page': 1  # Página de resultados
}

# Realizar la solicitud HTTP a la API
response = requests.get(api_url, params=params)

# Verificar si la solicitud fue exitosa (código 200)
if response.status_code == 200:
    # Convertir la respuesta JSON en una lista de diccionarios
    data = response.json()

    # Crear un DataFrame de Pandas a partir de la lista de diccionarios
    df = pd.DataFrame(data)
    # Ahora puedes usar df para realizar análisis en los datos
    print(df.head())

    # Crear una lista con los IDs de las 10 primeras criptomonedas
    top_coins_id = [coin['id'] for coin in data]

    # Imprimir la lista de IDs
    print(top_coins_id)
else:
    print('Error al obtener los datos de la API')


# %%
df.info()

# %%
df.head(10) # ordenado por capitalización de mercado

# %%
df.describe()


# %%
df.isnull().sum()

# %%
df.duplicated(subset=["current_price"]).sum()

# %%
df["fully_diluted_valuation"].head(10)

# %%



columns_to_remove = [3, 6, 9,10,11,12,13,14,  19, 20, 22,23, 25]  # Índices de las columnas a eliminar

# Eliminar las columnas usando el método drop con axis=1
df = df.drop(df.columns[columns_to_remove], axis=1)

# Ahora df solo contendrá las columnas restantes


# %%
df.info()  #df mercado global x capitalizacion de mercado

# %%
# Exportar el DataFrame a un archivo CSV
nombre_archivo = "df_markets_mark_cap.csv"
df.to_csv(nombre_archivo, index=False)

# %%
df

# %%

#df global x precio más alto
df_precios = df.sort_values(by='current_price', ascending=False)




# %%
df_precios.head(10)

# %%

#Exploramos datos historicos en la API, por moneda
# ID de la criptomoneda en CoinGecko (por ejemplo, 'bitcoin')
crypto_id = 'bitcoin'

# Fecha de inicio (1 de enero de 2023)
start_date = datetime(2023, 1, 1)  # Cambio aquí: Utilizar datetime en lugar de date

# Fecha de fin (hasta el último día de hoy)
end_date = datetime.now()

# Convertir las fechas a formato UNIX timestamp (en segundos)
start_timestamp = int(start_date.timestamp())
end_timestamp = int(end_date.timestamp())

# URL de la API de CoinGecko para obtener datos históricos en el rango de tiempo
api_url = f'https://api.coingecko.com/api/v3/coins/{crypto_id}/market_chart/range?vs_currency=usd&from={start_timestamp}&to={end_timestamp}'

# Realizar la solicitud HTTP a la API
response = requests.get(api_url)

# Verificar si la solicitud fue exitosa (código 200)
if response.status_code == 200:
    # Convertir la respuesta JSON en un diccionario
    data = response.json()

    # Extraer los datos de precios y fechas del diccionario
    prices = data['prices']
    timestamps = [timestamp for timestamp, _ in prices]

    # Crear un DataFrame de Pandas a partir de los datos
    df_historical = pd.DataFrame(prices, columns=['timestamp', 'price'])

    # Convertir los timestamps a fechas legibles
    df_historical['timestamp'] = pd.to_datetime(timestamps, unit='ms')

    # Ahora puedes usar df_historical para analizar los datos históricos de precios
    print(df_historical.head())
else:
    print('Error al obtener los datos históricos desde la API')


# %%

#Creamos una función para traer datos históricos de las 10 monedas principales
# Lista de los ID de las 10 principales criptomonedas
#Este codigo funciona, pero a veces cuando se vuelve a ejecutar falla para algunas monedas, y hay que volver a ejecutarlo hasta que de ok
top_coins_ids = [
    'bitcoin', 'ethereum', 'cardano', 'binancecoin', 'solana',
    'ripple', 'polkadot', 'dogecoin', 'avalanche-2', 'shiba-inu'
]

# Función para obtener datos históricos de una moneda
def get_historical_data(crypto_id):
    start_date = datetime(2023, 1, 1)
    end_date = datetime.now()

    start_timestamp = int(start_date.timestamp())
    end_timestamp = int(end_date.timestamp())

    api_url = f'https://api.coingecko.com/api/v3/coins/{crypto_id}/market_chart/range?vs_currency=usd&from={start_timestamp}&to={end_timestamp}'

    response = requests.get(api_url)

    if response.status_code == 200:
        data = response.json()

        prices = data['prices']
        timestamps = [timestamp for timestamp, _ in prices]
        market_caps = data['market_caps']
        total_volumes = data['total_volumes']

        df_historical = pd.DataFrame({
            'timestamp': pd.to_datetime(timestamps, unit='ms'),
            'price': [price for _, price in prices],
            'market_cap': [cap for _, cap in market_caps],
            'total_volume': [volume for _, volume in total_volumes],
            
        })

        return df_historical
    else:
        print(f'Error al obtener los datos históricos para {crypto_id}')
        return None

# Crear un diccionario para almacenar los DataFrames de cada moneda
dataframes_by_coin = {}

# Iterar a través de los IDs de moneda y obtener los datos históricos
for coin_id in top_coins_ids:
    dataframe = get_historical_data(coin_id)
    if dataframe is not None:
        dataframes_by_coin[coin_id] = dataframe


# %%
dataframes_by_coin["ethereum"]

# %%
# Imprimir el encabezado de los DataFrames generados de las top 10
for coin_id, dataframe in dataframes_by_coin.items():
    print(f"DataFrame para {coin_id}:")
    print(dataframe.head())
    print()

# %%
#Se busca hacer un merge de los dataframe para tener en un dataframe la series de tiempo de las 10 monedas ya realizar comparaciones
#Unificamos a raves de la columna de fecha y tomamos los pecios como variabla a analizar de cada moneda

# %%
dataframes_by_coin["bitcoin"]["price"]

# %%
#renombro columnas del dataframe para poder hacer el merge a través de timestamp
dataframes_by_coin["bitcoin"].rename(columns={"price": "bitcoin_price"}, inplace=True)
dataframes_by_coin["ethereum"].rename(columns={"price": "ethereum_price"}, inplace=True)
dataframes_by_coin["cardano"].rename(columns={"price": "cardano_price"}, inplace=True)
dataframes_by_coin["binancecoin"].rename(columns={"price": "binancecoin_price"}, inplace=True)
dataframes_by_coin["solana"].rename(columns={"price": "solana_price"}, inplace=True)
dataframes_by_coin["ripple"].rename(columns={"price": "ripple_price"}, inplace=True)
dataframes_by_coin["polkadot"].rename(columns={"price": "polkadot_price"}, inplace=True)
dataframes_by_coin["dogecoin"].rename(columns={"price": "dogecoin_price"}, inplace=True)
dataframes_by_coin["avalanche-2"].rename(columns={"price": "avalanche-2_price"}, inplace=True)
dataframes_by_coin["shiba-inu"].rename(columns={"price": "shiba-inu_price"}, inplace=True) 


# %%
dataframes_by_coin["bitcoin"].rename(columns={"market_cap": "bitcoin_market_cap"}, inplace=True)
dataframes_by_coin["ethereum"].rename(columns={"market_cap": "ethereum_market_cap"}, inplace=True)
dataframes_by_coin["cardano"].rename(columns={"market_cap": "cardano_market_cap"}, inplace=True)
dataframes_by_coin["binancecoin"].rename(columns={"market_cap": "binancecoin_market_cap"}, inplace=True)
dataframes_by_coin["solana"].rename(columns={"market_cap": "solana_market_cap"}, inplace=True)
dataframes_by_coin["ripple"].rename(columns={"market_cap": "ripple_market_cap"}, inplace=True)
dataframes_by_coin["polkadot"].rename(columns={"market_cap": "polkadot_market_cap"}, inplace=True)
dataframes_by_coin["dogecoin"].rename(columns={"market_cap": "dogecoin_market_cap"}, inplace=True)
dataframes_by_coin["avalanche-2"].rename(columns={"market_cap": "avalanche-2_market_cap"}, inplace=True)
dataframes_by_coin["shiba-inu"].rename(columns={"market_cap": "shiba-inu_market_cap"}, inplace=True) 

# %%
dataframes_by_coin["bitcoin"].rename(columns={"total_volume": "bitcoin_total_volume"}, inplace=True)
dataframes_by_coin["ethereum"].rename(columns={"total_volume": "ethereum_total_volume"}, inplace=True)
dataframes_by_coin["cardano"].rename(columns={"total_volume": "cardano_total_volume"}, inplace=True)
dataframes_by_coin["binancecoin"].rename(columns={"total_volume": "binancecoin_total_volume"}, inplace=True)
dataframes_by_coin["solana"].rename(columns={"total_volume": "solana_total_volume"}, inplace=True)
dataframes_by_coin["ripple"].rename(columns={"total_volume": "ripple_total_volume"}, inplace=True)
dataframes_by_coin["polkadot"].rename(columns={"total_volume": "polkadot_total_volume"}, inplace=True)
dataframes_by_coin["dogecoin"].rename(columns={"total_volume": "dogecoin_total_volume"}, inplace=True)
dataframes_by_coin["avalanche-2"].rename(columns={"total_volume": "avalanche-2_total_volume"}, inplace=True)
dataframes_by_coin["shiba-inu"].rename(columns={"total_volume": "shiba-inu_total_volume"}, inplace=True) 

# %% [markdown]
# Tomamos una lista de DataFrames que contienen información de precios de criptomonedas, los combinamos en un único DataFrame basado en la columna de tiempo ('timestamp'), realizamos algunas transformaciones, como renombrar la columna ultima a date y seleccionamos en los datos y finalmente producimos tres DataFrame finales, siendo el mas relvante para analisis el dataframe combinasdo con los precios de criptomonedass seleccionadas. 

# %%
#Hacemos el merge de precios
dataframes_list = [
    dataframes_by_coin["bitcoin"],
    dataframes_by_coin["ethereum"],
    dataframes_by_coin["cardano"],
    dataframes_by_coin["binancecoin"],
    dataframes_by_coin["solana"],
    dataframes_by_coin["ripple"],
    dataframes_by_coin["polkadot"],
    dataframes_by_coin["dogecoin"],
    dataframes_by_coin["avalanche-2"],
    dataframes_by_coin["shiba-inu"]
]

# Usar reduce para unir los DataFrames en la lista basados en el timestamp
from functools import reduce

# Función para unir dos DataFrames en función del timestamp
def merge_df(left, right):
    return pd.merge(left, right, on='timestamp', how='outer')

# Combinar todos los DataFrames en uno solo
combined_df = reduce(merge_df, dataframes_list)

# Ordenar el DataFrame por timestamp
combined_df = combined_df.sort_values(by='timestamp')

# Renombrar la columna "timestamp" a "date"
combined_df.rename(columns={'timestamp': 'date'}, inplace=True)

# Seleccionar las columnas que deseas mantener en el DataFrame combinado
columns_to_keep = ['date', 'bitcoin_price', 'ethereum_price', 'cardano_price', 'binancecoin_price', 'solana_price', 'ripple_price', 'polkadot_price', 'dogecoin_price', 'avalanche-2_price', 'shiba-inu_price']
combined_prices_df = combined_df[columns_to_keep]

columns_to_keep1 = ['date', 'bitcoin_market_cap', 'ethereum_market_cap', 'cardano_market_cap', 'binancecoin_market_cap', 'solana_market_cap', 'ripple_market_cap', 'polkadot_market_cap', 'dogecoin_market_cap', 'avalanche-2_market_cap', 'shiba-inu_market_cap']
combined_market_cap_df = combined_df[columns_to_keep1]

columns_to_keep2 = ['date', 'bitcoin_total_volume', 'ethereum_total_volume', 'cardano_total_volume', 'binancecoin_total_volume', 'solana_total_volume', 'ripple_total_volume', 'polkadot_total_volume', 'dogecoin_total_volume', 'avalanche-2_total_volume', 'shiba-inu_total_volume']
combined_total_volumne = combined_df[columns_to_keep2]

combined_prices_df.head()


# %%
combined_prices_df.describe()

# %%
# Exportar el DataFrame a un archivo CSV
nombre_archivo = "df_combined_prices.csv"
df.to_csv(nombre_archivo, index=False)

# %%
#importar archivo
archivo_csv = 'df_combined_prices.csv'  # Cambia esto a la ubicación real de tu archivo

# Carga el archivo CSV en un DataFrame
df_combined_prices = pd.read_csv(archivo_csv)

# %%
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns


# %%
plt.figure(figsize=(12, 6))
sns.lineplot(data=combined_prices_df, x='date', y='bitcoin_price', label='Bitcoin')
sns.lineplot(data=combined_prices_df, x='date', y='ethereum_price', label='Ethereum')
sns.lineplot(data=combined_prices_df, x='date', y='cardano_price', label='Cardano')
sns.lineplot(data=combined_prices_df, x='date', y='binancecoin_price', label='Binancecoin')
sns.lineplot(data=combined_prices_df, x='date', y='solana_price', label='Solana')
sns.lineplot(data=combined_prices_df, x='date', y='ripple_price', label='Ripple')
sns.lineplot(data=combined_prices_df, x='date', y='polkadot_price', label='Polkadot')
sns.lineplot(data=combined_prices_df, x='date', y='dogecoin_price', label='Dogecoin')
sns.lineplot(data=combined_prices_df, x='date', y='avalanche-2_price', label='Avalanche-2')
sns.lineplot(data=combined_prices_df, x='date', y='shiba-inu_price', label='Shiba_inu')

plt.xlabel('Año 2023')
plt.ylabel('Precio')
plt.title('Evolución de precios de las Top 10 criptomonedass de mayor capitalización')
plt.legend()
plt.show()


# %% [markdown]
# Observamos que hay escalas muy diferentes entre las criptomonedas, sobre todo el Bitcoin se diferencias mucho del resto. Vamos a ormalizalizar variables. Pero antes calculamos distribucion de frecuencias de cada uno, para saber por donde se ubico el precio con mayor precuencia durante 2023

# %%



# Supongamos que tienes combined_prices_df y las columnas de precios

# Lista de criptomonedas
crypto_columns = ['bitcoin_price', 'ethereum_price', 'cardano_price', 'binancecoin_price', 'solana_price',
                  'ripple_price', 'polkadot_price', 'dogecoin_price', 'avalanche-2_price', 'shiba-inu_price']

# Tamaño de la figura
fig, axes = plt.subplots(2, 5, figsize=(15, 10))
axes = axes.flatten()

# Iterar a través de las criptomonedas y crear histogramas
for i, crypto in enumerate(crypto_columns):
    ax = axes[i]
    sns.histplot(data=combined_prices_df, x=crypto, ax=ax, kde=True)
    ax.set_xlabel('Precio')
    ax.set_ylabel('Frecuencia')
    ax.set_title(crypto.replace("_", " ").title())

# Ajustar los subplots y añadir título
plt.tight_layout()
plt.suptitle('Distribución de Precios de las Top 10 Criptomonedas de Mayor Capitalización', y=1.02)
plt.show()


# %%
# Guardar la figura en un archivo de imagen (por ejemplo, PNG)
plt.savefig('histogramas_Top_10_criptomonedas.png')

# %%
#Buscamos detectar outliers con boxplot.
# Lista de criptomonedas
crypto_columns = ['bitcoin_price', 'ethereum_price', 'cardano_price', 'binancecoin_price', 'solana_price',
                  'ripple_price', 'polkadot_price', 'dogecoin_price', 'avalanche-2_price', 'shiba-inu_price']

# Tamaño de la figura
fig, axes = plt.subplots(2, 5, figsize=(15, 10))
axes = axes.flatten()

# Iterar a través de las criptomonedas y crear box plots
for i, crypto in enumerate(crypto_columns):
    ax = axes[i]
    sns.boxplot(data=combined_prices_df, y=crypto, ax=ax)
    ax.set_ylabel('Precio')
    ax.set_title(crypto.replace("_", " ").title())

# Ajustar los subplots y añadir título
plt.tight_layout()
plt.suptitle('Box Plots de Precios de las Top 10 Criptomonedas de Mayor Capitalización', y=1.02)

# Mostrar el gráfico en pantalla
plt.show()


# %%


# %%
#Ver los outliers de Ethereum, Solana, y Ripple
# Por el grafico vemos que el umbral de ethereum es 1300
umbral_ethereum = 1300

# Filtrar los valores atípicos de ethereum
outliers_ethereum = combined_prices_df[combined_prices_df["ethereum_price"] < umbral_ethereum]

# Contar la cantidad de outliers de ethereum
num_outliers_ethereum = len(outliers_ethereum)
print("Número de outliers de Ethereum:", num_outliers_ethereum)



# %%
outliers_ethereum.head(8)

# %%
#Ver los outliers de Solana

# Por el grafico vemos que el umbral de ethereum es 1300
umbral_solana = 15

# Filtrar los valores atípicos de ethereum
outliers_solana = combined_prices_df[combined_prices_df["solana_price"] < umbral_solana]

# Contar la cantidad de outliers de ethereum
num_outliers_solana = len(outliers_solana)
print("Número de outliers de Solana", num_outliers_solana)


# %%
outliers_solana.head(7)

# %%
#Ver los outliers de Ripple

# Por el grafico vemos que el umbral de Ripple es 1300
umbral_ripple = 0.69

# Filtrar los valores atípicos de ethereum
outliers_ripple = combined_prices_df[combined_prices_df["ripple_price"] > umbral_ripple]

# Contar la cantidad de outliers de ethereum
num_outliers_ripple = len(outliers_ripple)
print("Número de outliers de Ripple", num_outliers_ripple)


# %%
outliers_ripple.head(20)


# %% [markdown]
# En cuanto a los outliers, vemos que tanto Ethereum como Ripple, tienen 8 y 7, respectivamente, que marcan los minimos precios de principios de enero, y luego el mercado entró en una recuperación para 2023. En cambio Ripple, esta márcando outliers en sus máximos a fines de julio y agosto, dando cuenta de que en entre fines de julio y agosto un una fuerte alza de precios desde fine de julio y siguió en agosto.Eventualmente podemos quitar los outliers para calcular el precio promedio del año. Dejamos los outliers de Ethereum y Solana porque son pocos, para mantener la integridad temporal y poder efectuar un analisis comparativo con el rresto de las monedas. En el caso de Ripple los dejamos porque justamente el análisis marca que una aceleración alcista reciente de las moneda.

# %% [markdown]
# Noticias Ripple, en agosto Ripple obtvo victoria parcial ante la sec y animo a inversorres a seguir invirtiendo. informes recientes señala que :
# -Los inversores institucionales han invertido en Ripple (XRP) durante 16 semanas consecutivas según un informe de CoinShares.
# -Los activos bajo gestión de productos de inversión en XRP han aumentado un 127% desde principios de año.
# -Se han invertido mas de $11,25 millones en XRP desde comienzos de 2023.
# 
# https://decrypt.co/es/152465/inversores-institucionales-apuestan-fuertemente-por-xrp-tras-victoria-parcial-de-ripple-vs-la-sec

# %%
combined_prices_df.describe()

# %%
from sklearn.preprocessing import MinMaxScaler
# Extraer las fechas antes de la normalización
dates = combined_prices_df['date']

# Normalizar los precios de las criptomonedas (excluyendo la columna de fechas)
scaler = MinMaxScaler()
normalized_prices = scaler.fit_transform(combined_prices_df.drop('date', axis=1))

# Crear un nuevo DataFrame normalizado
nor_combined_prices_df = pd.DataFrame(normalized_prices, columns=combined_prices_df.columns[1:])

# Agregar las fechas de nuevo al DataFrame normalizado
nor_combined_prices_df.insert(0, 'date', dates)

# %%
plt.figure(figsize=(14, 8))
sns.lineplot(data=nor_combined_prices_df, x='date', y='bitcoin_price', label='Bitcoin')
sns.lineplot(data=nor_combined_prices_df, x='date', y='ethereum_price', label='Ethereum')
sns.lineplot(data=nor_combined_prices_df, x='date', y='cardano_price', label='Cardano')
sns.lineplot(data=nor_combined_prices_df, x='date', y='binancecoin_price', label='Binancecoin')
sns.lineplot(data=nor_combined_prices_df, x='date', y='solana_price', label='Solana')
sns.lineplot(data=nor_combined_prices_df, x='date', y='ripple_price', label='Ripple')
sns.lineplot(data=nor_combined_prices_df, x='date', y='polkadot_price', label='Polkadot')
sns.lineplot(data=nor_combined_prices_df, x='date', y='dogecoin_price', label='Dogecoin')
sns.lineplot(data=nor_combined_prices_df, x='date', y='avalanche-2_price', label='Avalanche-2')
sns.lineplot(data=nor_combined_prices_df, x='date', y='shiba-inu_price', label='Shiba_inu')

plt.xlabel('Año 2023')
plt.ylabel('Precio')
plt.title('Evolución de precios de las Top 10 criptomonedass de mayor capitalización')
plt.legend()
plt.show()


# %%
# Guardar la figura en un archivo de imagen (por ejemplo, PNG)
plt.savefig('Evolución del precio_Top_10_criptomonedas.png')

# %%
 
# Convertir la columna de fecha a formato datetime
combined_prices_df.loc[:,'date'] = pd.to_datetime(combined_prices_df['date'])

# Definir la lista de criptomonedas para facilitar el análisis
cryptocurrencies = combined_prices_df.columns[1:]

# Scatter plot para visualizar las relaciones entre criptomonedas
plt.figure(figsize=(10, 8))
sns.pairplot(combined_prices_df, x_vars=cryptocurrencies, y_vars=cryptocurrencies)
plt.title("Relaciones entre las criptomonedas")
plt.show()



plt.savefig('Scatterplot_Top_10_criptomonedas.png')
# Cálculo del ROI
#initial_investment = 10000  # Cambia el valor según tu inversión inicial
#final_portfolio_value = combined_prices_df.iloc[-1, 1:]  # Última fila de precios
#roi = (final_portfolio_value / initial_investment - 1) * 100
#print("ROI por criptomoneda:\n", roi)




# %% [markdown]
# Tanto en el scatterplot como en la matriz de correlación se observa que Bitcoin y Ethereum tienen una alta correlacón positiva. Cuando Bitcoin sube, también sube Ethereum, y lo mismo cuando el sentimiento es negativo hacia estas monedas, cuando baja una, también baja la otra. Luego, se ubican cardano, binance, y dogcoin, avalanche-2 y polkadot con uan fuerte correlacion positiva. Estos es, cuando una de ellas se incrementa, tambien se incrementa la otra, aunque la correlación no implique que el ascenso de una moneda sea la causa de la otra. 

# %%
#Correlaciones entre variables
correlation_matrix = combined_prices_df.corr()
plt.figure(figsize=(10, 8))
sns.heatmap(correlation_matrix, annot=True, cmap='coolwarm')
plt.title("Matriz de Correlación")
plt.show()
plt.savefig('Correlación_Top_10_criptomonedas.png')




# %%


# %%



# Obtener la primera y última fecha en el dataframe
first_date = combined_prices_df['date'].iloc[0]
last_date = combined_prices_df['date'].iloc[-1]

# Filtrar el dataframe por la primera y última fecha
first_price_df = combined_prices_df[combined_prices_df['date'] == first_date]
last_price_df = combined_prices_df[combined_prices_df['date'] == last_date]

# Calcular la variación del precio para cada criptomoneda
price_columns = ['bitcoin_price', 'ethereum_price', 'cardano_price', 'binancecoin_price',
'ripple_price', 'polkadot_price', 'dogecoin_price', 'avalanche-2_price']

for crypto in price_columns:
    first_price = first_price_df[crypto].values[0]
    last_price = last_price_df[crypto].values[0]

    
    relative_change = ((last_price - first_price) / first_price) * 100
    print(f"Variación de {crypto} durante lo que va del año 2023:", round(relative_change,2), "%")

    
    




# %% [markdown]
# Entre las criptomoedas de mayor capitalización, durante el 2023 las monedas de mejor desempeño resultaron el Bitcoin (59,49%),  Ripple(48,8%, y Ethereum(39,58%). Vamos a tomar estas tres para acotar el análisis y compararlas con el desempeño de las acciones de Estados Unidos. Elegimos el SPY, que es el ticker y el nombre coloquial del derivado (ETF) que emula el comportamiento del S&P 500, el índice que mira el comportamiento de las 500 mayores empresas de la Economía de EEUU. Es un buen indicador para medir la salud de las economía y las empresas en general, que a su vez están muy afectadas por la inflación e inflación esperada, que se observa en las políticas que toma la Reserva Federal de los Estados Unidos, a través de la política monetaria y en particular de los movicmientos en las tasa de interés de referencia. Esto afecta particularmente el mercado de criptomonedas, ya que si se espera que la economía de EEUU empeore, en el sentido de que suba la inflación y la FED se vea obligada a subir las tasas, habrá más retracción en la inversión y  fuga de capitales hacia activos considerados más seguros, que tendrán un rendimiento mayor, y son considerados y menos volátiles. Por el contrario, si hay mejora en la marcha de las empresas y de la saluda de la economía en general, con una baja inflación, la FED hará la política contraria, expandirá más dinero en la economía para invertir, y las criptomonedas tendrán mayor margen de aceptación enre los inversores.    

# %%
import pandas as pd

# Cambia la ruta del archivo si es necesario
spy = "SPY__S&P_500_2023.csv"

# Utiliza pandas para leer el archivo CSV
df_spy = pd.read_csv(spy)

# Ahora puedes trabajar con los datos
df_spy.head()  # Imprime las primeras filas del DataFrame


# %%
# Renombra la columna "date" a "new_date" (cambia "new_date" al nombre deseado)
df_spy.rename(columns={"Date": "date", "Price":"price"}, inplace=True)

# %%
# Selecciona las columnas "date" y "price"
df_spy_price = df_spy[["date", "price"]]

# %%
type(df_spy_price["date"])

# %%
type(combined_prices_df["date"])

# %%
import pandas as pd
from functools import reduce

# Supongamos que df_spy_price tiene la columna "date" en formato de cadena
# y combined_prices_df tiene la columna "date" en formato datetime64[ns]

# Convertir la columna "date" en df_spy_price a datetime64[ns] usando .loc[]
df_spy_price.loc[:, 'date'] = pd.to_datetime(df_spy_price['date'])

combined_prices_df.loc[:,"date"]= pd.to_datetime(combined_prices_df['date'])


# %%
print(combined_prices_df.columns)
print(df_spy_price.columns)


# %%
print(combined_prices_df['date'].dtype)
print(df_spy_price['date'].dtype)


# %%
df_spy_price.loc[:, 'date'] = pd.to_datetime(df_spy_price['date'])


# %%
# Luego proceder con la combinación
dataframes_list1 = [combined_prices_df, df_spy_price]

def merge_df(left, right):
    return pd.merge(left, right, on='date', how='outer')

combined_prices_spy_df = reduce(merge_df, dataframes_list1)
combined_prices_spy_df = combined_prices_spy_df.sort_values(by='date')

combined_prices_spy_df.head()

# %%
# Renombrar la columna 'price' a 'spy_price'
combined_prices_spy_df.rename(columns={'price': 'spy_price'}, inplace=True)

# Contar los valores nulos en la columna 'spy_price'
n_nulls = combined_prices_spy_df['spy_price'].isnull().sum()
print(f"Número de valores nulos en spy_price: {n_nulls}")




# %%


# %%
import datetime

def count_weekdays(year, day_of_week):
    count = 0
    start_date = datetime.date(year, 1, 1)
    end_date = datetime.date(year, 8, 19)
    
    delta = datetime.timedelta(days=1)
    current_date = start_date
    
    while current_date <= end_date:
        if current_date.weekday() == day_of_week:
            count += 1
        current_date += delta
    
    return count

year = 2023  # Cambia esto al año que desees
saturdays = count_weekdays(year, 5)  # 5 representa el sábado
sundays = count_weekdays(year, 6)  # 6 representa el domingo

print(f"En el año {year} hay {saturdays} sábados y {sundays} domingos.")


# %%
#Llenar los valores nulos en 'spy_price' con el valor del día anterior
combined_prices_spy_df['spy_price'].fillna(method='ffill', inplace=True)

# %%
combined_prices_spy_df.head(50)

# %%
# Asigno valor al registro cero que aun quedó con NaN
valor_x = 561.35


combined_prices_spy_df.loc[0, 'spy_price'] = valor_x



# %%

# Exportar el DataFrame a un archivo CSV
nombre_archivo = "combined_prices_spy_df.csv"
df.to_csv(nombre_archivo, index=False)

# %%
import seaborn as sns
import matplotlib.pyplot as plt

# Divide los precios por el primer valor para indexarlos
combined_prices_spy_df['bitcoin_price'] = combined_prices_spy_df['bitcoin_price'] / combined_prices_spy_df['bitcoin_price'].iloc[0]
combined_prices_spy_df['spy_price'] = combined_prices_spy_df['spy_price'] / combined_prices_spy_df['spy_price'].iloc[0]
combined_prices_spy_df['ethereum_price'] = combined_prices_spy_df['ethereum_price'] / combined_prices_spy_df['ethereum_price'].iloc[0]
combined_prices_spy_df['ripple_price'] = combined_prices_spy_df['ripple_price'] / combined_prices_spy_df['ripple_price'].iloc[0]

# Utiliza Seaborn para crear el gráfico de líneas comparando los precios indexados
sns.set(style='darkgrid')
sns.lineplot(data=combined_prices_spy_df[['bitcoin_price', 'ethereum_price', 'ripple_price', 'spy_price']])
plt.xlabel('Fecha') 
plt.ylabel('Precio Indexado')
plt.suptitle('Variación de precios en Bitcoin, Ethereum y Ripple vs SPY',  y=1.02, fontsize=16)
plt.title('Comparación de precios con precios indexados a 1')
plt.savefig('Precios_TOP3_cripto_SPY.png')
plt.show()



# %%


# %%
#Confirmamos que spy nno tenga valores nuleo
null_count = combined_prices_spy_df['spy_price'].isnull().sum()
print(f"Número de valores nulos en spy_price: {null_count}")


# %%
#Calculo ls variaciones de precios, en lo que va de 2023
# Obtener la primera y última fecha en el dataframe
first_date = combined_prices_spy_df['date'].iloc[0]
last_date = combined_prices_spy_df['date'].iloc[-1]

# Filtrar el dataframe por la primera y última fecha
first_price_df = combined_prices_spy_df[combined_prices_df['date'] == first_date]
last_price_df = combined_prices_spy_df[combined_prices_df['date'] == last_date]

# Calcular la variación del precio para cada criptomoneda
price_columns = ['bitcoin_price', 'ethereum_price', 'cardano_price', 'binancecoin_price',
'ripple_price', 'polkadot_price', 'dogecoin_price', 'avalanche-2_price', 'spy_price']

for crypto in price_columns:
    first_price = first_price_df[crypto].values[0]
    last_price = last_price_df[crypto].values[0]

    
    relative_change = ((last_price - first_price) / first_price) * 100
    print(f"Variación de {crypto} durante lo que va del año 2023:", round(relative_change,2), "%")


# %%
import pandas as pd
import matplotlib.pyplot as plt
import ta

# Supongamos que combined_prices_df es tu DataFrame con los datos
# y crypto_columns es la lista de columnas de precios de criptomonedas
# Calcular las Medias Móviles Exponenciales (EMA)
for column in crypto_columns:
    combined_prices_spy_df[f'{column}_EMA20'] = combined_prices_spy_df[column].ewm(span=20, adjust=False).mean()

# Calcular las Medias Móviles Exponenciales (EMA) para SPY
combined_prices_spy_df['spy_EMA20'] = combined_prices_spy_df['spy_price'].ewm(span=20, adjust=False).mean()

# Graficar los datos para criptomonedas y SPY
for column in crypto_columns:
    fig, axes = plt.subplots(nrows=1, figsize=(10, 4))
    fig.suptitle(f'Análisis técnico para {column}', fontsize=16)

    axes.plot(combined_prices_spy_df[column], label='Precio de cierre', color='blue')
    axes.plot(combined_prices_spy_df[f'{column}_EMA20'], label='EMA 20', color='orange')
    axes.plot(combined_prices_spy_df['spy_price'], label='Precio SPY', color='green')  # Agregar SPY
    axes.plot(combined_prices_spy_df['spy_EMA20'], label='EMA 20 SPY', color='red')  # Agregar EMA SPY
    axes.set_ylabel('Precio')
    axes.legend()

    plt.tight_layout()
    plt.show()


# %%
import pandas as pd
import matplotlib.pyplot as plt
import ta

# 
#  crypto_columns es la lista de columnas de precios de criptomonedas
# Calcular las Medias Móviles Exponenciales (EMA)
for column in crypto_columns:
    combined_prices_spy_df[f'{column}_EMA20'] = combined_prices_spy_df[column].ewm(span=20, adjust=False).mean()

# Calcular las Medias Móviles Exponenciales (EMA) para SPY
    combined_prices_spy_df['spy_EMA20'] = combined_prices_spy_df['spy_price'].ewm(span=20, adjust=False).mean()

# Graficar los datos
for column in crypto_columns:
    fig, axes = plt.subplots(nrows=1, figsize=(10, 4))
    fig.suptitle(f'Análisis técnico para {column}', fontsize=16)

    axes.plot(combined_prices_spy_df[column], label='Precio de cierre', color='blue')
    axes.plot(combined_prices_spy_df[f'{column}_EMA20'], label='EMA 20', color='orange')
    axes.set_ylabel('Precio')
    axes.legend()

    plt.tight_layout()
    plt.show()


# %%
import pandas as pd
import matplotlib.pyplot as plt
import ta

# Supongamos que combined_prices_df es tu DataFrame con los datos
# y crypto_columns es la lista de columnas de precios de criptomonedas

# Calcular las Medias Móviles Exponenciales (EMA)
for column in crypto_columns:
    combined_prices_spy_df[f'{column}_EMA20'] = combined_prices_spy_df[column].ewm(span=20, adjust=False).mean()

# Dividir las criptomonedas en grupos de 5
crypto_groups = [crypto_columns[i:i+5] for i in range(0, len(crypto_columns), 5)]

# Graficar los datos (5 gráficos por fila)
for group in crypto_groups:
    fig, axes = plt.subplots(nrows=1, ncols=len(group), figsize=(5*len(group), 4))
    
    for i, column in enumerate(group):
        axes[i].plot(combined_prices_spy_df[column], label='Precio de cierre', color='blue')
        axes[i].plot(combined_prices_spy_df[f'{column}_EMA20'], label='EMA 20', color='orange')
        axes[i].set_ylabel('Precio')
        axes[i].set_title(f'Análisis técnico EMA20 para {column}')
        axes[i].legend()


    plt.tight_layout()
    plt.show()


# %% [markdown]
# La EMA20 (Exponential Moving Average 20) calcula la media móvil exponencial de los últimos 20 días. En otras palabras, toma en cuenta los precios de los últimos 20 días, otorgando mayor peso a los datos más recientes y reduciendo gradualmente el peso de los datos más antiguos.
# Cuando se calcula la EMA20, el énfasis está en los datos más recientes, lo que significa que los precios más recientes tienen un impacto mayor en el valor de la EMA20 en comparación con los precios más antiguos. Esto la hace más sensible a los cambios recientes en los precios y puede ayudar a capturar tendencias emergentes o cambios en la dirección del mercado más rápidamente que una media móvil simple (SMA).
# En resumen, la EMA20 calcula la media de los precios de los últimos 20 días, con mayor peso en los datos más recientes, y se utiliza comúnmente en el análisis técnico para evaluar la dirección de la tendencia y los posibles puntos de inversión.
# La posición relativa del EMA 20 con respecto al precio de cierre en un gráfico puede proporcionar información sobre la dirección y fuerza de la tendencia de un activo financiero, como una criptomoneda. Aquí hay algunas pautas generales:
# EMA 20 Por Encima del Precio de Cierre: Si el EMA 20 está por encima del precio de cierre, puede indicar una tendencia alcista o positiva. Esto sugiere que la tendencia general de los precios es ascendente y el activo podría estar ganando fuerza. Los inversores a menudo lo consideran un signo positivo y pueden interpretarlo como una señal de compra.
# EMA 20 Por Debajo del Precio de Cierre: Si el EMA 20 está por debajo del precio de cierre, puede indicar una tendencia bajista o negativa. Esto sugiere que la tendencia general de los precios es descendente y el activo podría estar perdiendo fuerza. Los inversores a menudo lo consideran un signo negativo y pueden interpretarlo como una señal de venta.
# Cruce de EMA y Precio de Cierre: Uno de los patrones técnicos comunes es el cruce entre el EMA y el precio de cierre. Un "cruce alcista" ocurre cuando el EMA cruza por encima del precio de cierre, lo que puede indicar un cambio potencial hacia una tendencia alcista. Un "cruce bajista" ocurre cuando el EMA cruza por debajo del precio de cierre, sugiriendo un cambio potencial hacia una tendencia bajista.
# 
# Es importante tener en cuenta que estas interpretaciones son indicativas y deben considerarse en el contexto de otros indicadores y análisis técnico más amplio. Además, los mercados financieros pueden ser volátiles y están sujetos a cambios rápidos, por lo que es esencial realizar análisis cuidadosos y considerar múltiples factores antes de tomar decisiones de inversión.

# %% [markdown]
# Applying The Fear and Greed Index in Investing
# Although investors are generally more interested in bitcoin’s Fear and Greed Index, there are high chances of sentiments varying across assets. Investors using the Fear and Greed Index should make a number of considerations, including; 
# 
# Asset of interest
# Bitcoin dominates the cryptocurrency space and dictates the movement of other assets to some extent. However, there are frequent cases of assets moving against bitcoin. Depending on the sentiments of the crypto community towards a particular asset, it could continue to go up even when bitcoin and the majority of crypto assets see a drop in value and sentiments. 
# 
# Unfortunately, the Fear and Greed Index metrics aren’t available for many assets. When available, this should be considered alongside that of bitcoin and any other asset that affects the concerned project. Relying solely on fear and greed metrics might lead to making wrong decisions.
# 
# Duration of investment
# The Fear and Greed Index might be more relevant in the short term. Investors who trade on a shorter-term timeframe should consider the periodic variation in general market sentiments before trading their assets or marking a purchase.

# %%
!pip install io

# %%
df_fear_and_greed.head()

# %%
data.head(10)

# %%
pip install plotly


# %%
import requests
import pandas as pd

url = "https://api.alternative.me/fng/?limit=10&format=csv&date_format=us"
response = requests.get(url)

if response.status_code == 200:
    data_lines = response.content.decode('utf-8').split('\n')[1:]
    
    data_list = []
    for line in data_lines:
        if line.strip():
            values = line.split(',')
            try:
                if len(values) >= 2:
                    data_dict = {
                        'time': values[0],
                        'value': float(values[1])
                    }
                    data_list.append(data_dict)
                else:
                    print("Skipping line due to missing values:", line)
            except ValueError as e:
                print(f"Error converting to float: {e}")
                print(f"Problematic line: {line}")
    
    df = pd.DataFrame(data_list)
    print(df)
else:
    print("Error al obtener los datos:", response.status_code)


# %%
import requests
import pandas as pd

start_date = "2023-01-01"
end_date = "2023-08-20"

url = f"https://api.alternative.me/fng/?limit=1000&format=csv&date_format=us&from={start_date}&to={end_date}"
response = requests.get(url)

if response.status_code == 200:
    data_lines = response.content.decode('utf-8').split('\n')[1:]
    
    data_list = []
    for line in data_lines:
        if line.strip():
            values = line.split(',')
            try:
                if len(values) >= 2:
                    data_dict = {
                        'time': values[0],
                        'value': float(values[1])
                    }
                    data_list.append(data_dict)
                else:
                    print("Skipping line due to missing values:", line)
            except ValueError as e:
                print(f"Error converting to float: {e}")
                print(f"Problematic line: {line}")
    
    df_FGI = pd.DataFrame(data_list)
    print(df_FGI)
else:
    print("Error al obtener los datos:", response.status_code)


# %%
df_FGI.head(10)

# %%
import requests
import pandas as pd
import matplotlib.pyplot as plt

start_date = "2023-01-01"
end_date = "2023-08-20"

url = f"https://api.alternative.me/fng/?limit=1000&format=csv&date_format=us&from={start_date}&to={end_date}"
response = requests.get(url)

if response.status_code == 200:
    data_lines = response.content.decode('utf-8').split('\n')[1:]
    
    data_list = []
    for line in data_lines:
        if line.strip():
            values = line.split(',')
            try:
                if len(values) >= 2:
                    data_dict = {
                        'time': values[0],
                        'value': float(values[1])
                    }
                    data_list.append(data_dict)
                else:
                    print("Skipping line due to missing values:", line)
            except ValueError as e:
                print(f"Error converting to float: {e}")
                print(f"Problematic line: {line}")
    
    df = pd.DataFrame(data_list)
    
    # Convertir 'time' a tipo fecha
    df['time'] = pd.to_datetime(df['time'])
    
    # Filtrar solo los datos de 2023
    df_2023 = df[(df['time'] >= '2023-01-01') & (df['time'] <= '2023-08-20')]
    
    # Graficar los datos
    plt.figure(figsize=(10, 6))
    plt.plot(df_2023['time'], df_2023['value'], marker='o', linestyle='-', color='b')
    plt.title('Fear and Greed Index - Año 2023')
    plt.xlabel('Fecha')
    plt.ylabel('Valor')
    plt.grid(True)
    plt.xticks(rotation=45)
    plt.tight_layout()
    plt.show()
else:
    print("Error al obtener los datos:", response.status_code)


# %%
<img src="https://alternative.me/crypto/fear-and-greed-index.png" alt="Latest Crypto Fear & Greed Index" />

# %%


url = "https://api.coingecko.com/api/v3/global"

response = requests.get(url)

if response.status_code == 200:
    data = response.json()
    
    # Crear un DataFrame a partir de los datos
    df_global = pd.DataFrame([data["data"]])
    
    # Imprime los datos globales
    print("Datos Globales:")
    print("Total de criptomonedas:", data["data"]["active_cryptocurrencies"])
    print("Total de mercados:", data["data"]["markets"])
    print("BTC Dominancia:", data["data"]["market_cap_percentage"]["btc"], "%")
    print("Volumen total 24h:", data["data"]["total_volume"]["usd"], "USD")
    print("Capitalización total:", data["data"]["total_market_cap"]["usd"], "USD")
else:
    print("Error al obtener los datos. Código de estado:", response.status_code)


# %%
df_global.head()

# %%
#traer api dolar blue, precios, o de cripto ya, para ver distintos dolares y exhchanges locales

# %%
url = "https://api.coingecko.com/api/v3/exchanges"

response = requests.get(url)

if response.status_code == 200:
    data = response.json()
    
    # Crear un DataFrame a partir de los datos
    df_exchanges = pd.DataFrame(data)
    print(df_exchanges)
else:
    print("Error en la solicitud a la API")


# %%
df_exchanges.head(10)

# %%

#Empresas que cotizan en bolsa que tienen criptomonedas entre sus activos

url = "https://api.coingecko.com/api/v3/companies/public_treasury/bitcoin"


response = requests.get(url)

if response.status_code == 200:
    data = response.json()
    
    # Crear un DataFrame a partir de los datos
    df_bolsa = pd.DataFrame(data)
    print(df_exchanges)
else:
    print("Error en la solicitud a la API")


# %%


# %%
df_bolsa.head() # ver como traer los datos de companies,,, ver arriba algun ej similar


