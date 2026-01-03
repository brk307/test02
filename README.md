import pandas as pd
import plotly.express as px
import os

POPULATION_FILE_NAME = "API_SP.POP.TOTL_DS2_en_csv_v2_2763937.csv"

#iki data arasi eslestirme
COUNTRY_NAME_MAP = {
    'United States': 'United States',
    'Czech Republic': 'Czechia',
    'South Korea': 'Korea, Rep.',
    'Taiwan Province of China': 'Taiwan',
    'Hong Kong S.A.R., China': 'Hong Kong SAR, China',
    'Russia': 'Russian Federation',
    'Iran': 'Iran, Islamic Rep.',
    'Venezuela': 'Venezuela, RB',
    'Egypt': 'Egypt, Arab Rep.',
    'Syria': 'Syrian Arab Republic',
    'Yemen': 'Yemen, Rep.',
    'Ivory Coast': "Cote d'Ivoire",
    'Congo (Brazzaville)': 'Congo, Rep.',
    'Congo (Kinshasa)': 'Congo, Dem. Rep.',
    'North Cyprus': 'Cyprus',
    'Palestinian Territories': 'West Bank and Gaza',
    'Somaliland region': 'Somalia',
    'Northern Cyprus': 'Cyprus',
    'Somaliland Region': 'Somalia'
}

#veri yukleme
def prepare_data():

    years = ['2015', '2016', '2017', '2018', '2019']     #mutluluk
    happy_dfs = []
    
    col_map = {
        'Country': 'Country', 'Country or region': 'Country',
        'Happiness Rank': 'Rank', 'Overall rank': 'Rank',
        'Happiness Score': 'Score', 'Score': 'Score',
        'Economy (GDP per Capita)': 'GDP', 'GDP per capita': 'GDP', 'Logged GDP per capita': 'GDP',
        'Region': 'Region',
        'Happiness.Score': 'Score', 'Economy..GDP.per.Capita.': 'GDP'
    }
    
    region_reference = {}
    
    for y in years:
        filename = f"{y}.csv"
        if os.path.exists(filename):
            try:
                df = pd.read_csv(filename, encoding='utf-8')
            except:
                df = pd.read_csv(filename, encoding='latin1')
            
            df.columns = df.columns.str.strip()
            df.rename(columns=col_map, inplace=True)
            df['Year'] = int(y)
            
            if 'Region' in df.columns:
                for _, row in df.iterrows():
                    if pd.notna(row['Country']) and row['Country'] not in region_reference:
                        region_reference[row['Country']] = row['Region']
            
            if 'Score' in df.columns and 'GDP' in df.columns:
                df = df[['Country', 'Year', 'Score', 'GDP']].copy()
                df['Country'] = df['Country'].str.strip()
                happy_dfs.append(df)
    
    if not happy_dfs:
        print("HATA: Hiçbir mutluluk dosyası yüklenemedi!")
        return None

    df_happiness = pd.concat(happy_dfs, ignore_index=True)
    df_happiness['Country'] = df_happiness['Country'].replace(COUNTRY_NAME_MAP)
    df_happiness['Region'] = df_happiness['Country'].map(region_reference).fillna('Other')

    if not os.path.exists(POPULATION_FILE_NAME):  #nufus
        print(f"HATA: {POPULATION_FILE_NAME} dosyası bulunamadı!")
        return None
    
    try:
        #direkt okuma 
        df_pop = pd.read_csv(POPULATION_FILE_NAME)
        df_pop.columns = [c.strip() for c in df_pop.columns]
        
        #ilk 4 satır kayınca okumuyorsa***
        if 'Country Name' not in df_pop.columns:
             df_pop = pd.read_csv(POPULATION_FILE_NAME, skiprows=4)

    except Exception as e:
        print(f"HATA: Nüfus dosyası okunamadı: {e}")
        return None

  
    df_pop.columns = [c.strip() for c in df_pop.columns]
    
    if 'Country Name' in df_pop.columns:
        df_pop.rename(columns={'Country Name': 'Country'}, inplace=True)
    
    year_cols = [c for c in df_pop.columns if str(c).isdigit() and int(c) >= 2015 and int(c) <= 2019]
    
    if not year_cols:
        print("HATA: Nüfus dosyasında 2015-2019 yılları bulunamadı!")
        return None
    
    df_pop = df_pop.melt(id_vars=['Country'], value_vars=year_cols, var_name='Year', value_name='Population')
    df_pop['Year'] = df_pop['Year'].astype(int)
    df_pop['Population'] = pd.to_numeric(df_pop['Population'], errors='coerce')

    #birlestirme
    df_final = pd.merge(df_happiness, df_pop, on=['Country', 'Year'], how='inner')
    df_final = df_final.dropna(subset=['Population', 'GDP', 'Score'])
    
    return df_final

#grafikk

def create_gapminder_chart(df):
    if df is None or df.empty:
        print("Veri hatası nedeniyle grafik çizilemedi!")
        return

    color_map = {
        'Western Europe': '#3b82f6', 'North America': '#10b981', 'Australia and New Zealand': '#f59e0b',
        'Middle East and Northern Africa': '#ec4899', 'Latin America and Caribbean': '#14b8a6',
        'Southeastern Asia': '#f97316', 'Central and Eastern Europe': '#8b5cf6', 'Eastern Asia': '#ef4444',
        'Sub-Saharan Africa': '#84cc16', 'Southern Asia': '#06b6d4', 'Other': '#94a3b8'
    }
    
    fig = px.scatter(
        df, x="GDP", y="Score", animation_frame="Year", animation_group="Country",
        size="Population", color="Region", hover_name="Country",
        hover_data={'GDP': ':.2f', 'Score': ':.2f', 'Population': ':,.0f', 'Region': True, 'Year': False},
        log_x=True, size_max=60, range_y=[2, 8.5],
        title="🌍 Dünya Mutluluk Raporu: GDP, Mutluluk ve Nüfus (2015-2019)",
        labels={'GDP': 'GDP per Capita (log scale)', 'Score': 'Happiness Score', 'Population': 'Nüfus', 'Region': 'Bölge'},
        template="plotly_white", color_discrete_map=color_map
    )
    
    fig.layout.updatemenus[0].buttons[0].args[1]["frame"]["duration"] = 1000
    fig.layout.updatemenus[0].buttons[0].args[1]["transition"]["duration"] = 500
    
    fig.show()

#run

if __name__ == "__main__":
    dataset = prepare_data()
    if dataset is not None:
        create_gapminder_chart(dataset)
