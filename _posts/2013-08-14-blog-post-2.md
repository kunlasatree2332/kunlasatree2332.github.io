---
title: 'Automating 7-Day Provincial Weather Forecasting'
date: 2025-12-11
permalink: /posts/weather-api-normalization
tags:
  - API Integration
---

Dealing with raw meteorological data often requires handling inconsistent JSON structures. The Thai Meteorological Department (TMD) API, for instance, returns forecast data as either dictionaries or lists depending on the timeframe. To analyze this effectively, we must first normalize the data into a standard list format using type-checking logic to prevent runtime errors during extraction.

By extracting the max_temp values for specific provinces, we can plot linear trends over a 7-day period. Furthermore, merging this forecast data with GeoJSON files allows for the creation of provincial heatmaps, visually identifying which regions in Thailand will experience peak temperatures in the coming week.

<img width="982" height="493" alt="image" src="https://github.com/user-attachments/assets/cf030d66-712c-49eb-b701-e1e4d4e88672" />

<img width="648" height="974" alt="image" src="https://github.com/user-attachments/assets/b5c03c56-c61b-45ae-91ab-6a41c3c4bfca" />



---python
import requests
import pandas as pd
import matplotlib.pyplot as plt
import geopandas as gpd

# ==========================================
# 1. API Configuration
# ==========================================
UID = "api"             # Replace with your actual UID
UKEY = "api12345"       # Replace with your actual UKEY
BASE_URL = "http://data.tmd.go.th/api"

def get_tmd_data(endpoint):
    url = f"{BASE_URL}/{endpoint}"
    params = {'uid': UID, 'ukey': UKEY, 'format': 'json'}
    try:
        # Added a slightly longer timeout just in case
        response = requests.get(url, params=params, timeout=20)
        return response.json()
    except Exception as e:
        print(f"Error fetching data: {e}")
        return None

def process_forecast_7days():
    print("\n--- Fetching 7-Day Weather Forecast (Final Fix) ---")
    data = get_tmd_data("WeatherForecast7Days/v2") 
    
    if not data: return

    # 1. Unwrap the outer layer
    if 'WeatherForecast7Days' in data:
        data = data['WeatherForecast7Days']

    # 2. Extract Provinces
    provinces = data.get('Provinces', {}).get('Province', [])
    # Handle single item return (dict) by converting to list
    if isinstance(provinces, dict): provinces = [provinces]
        
    print(f"Found data for {len(provinces)} provinces")

    map_data = []
    target_province = "Bangkok" 
    graph_dates = []
    graph_temps = []

    for prov in provinces:
        prov_name = prov.get('ProvinceNameEnglish', '') 
        forecast_data = prov.get('SevenDaysForecast', {})

        # ========================================================
        # CRITICAL FIX: Normalize data to a List regardless of input format
        # ========================================================
        daily_forecasts = []
        
        # Case A: Data received as a Dictionary (Column-oriented / Split arrays)
        # This matches the XML structure you encountered
        if isinstance(forecast_data, dict):
            # Extract lists of dates and temperatures
            dates = forecast_data.get('ForecastDate', [])
            temps = forecast_data.get('MaximumTemperature', [])
            
            # If there is only one day, API returns a String; convert to List
            if not isinstance(dates, list): dates = [dates]
            if not isinstance(temps, list): temps = [temps]

            # Zip them together to create a standard List of Dictionaries
            for d, t in zip(dates, temps):
                daily_forecasts.append({'date': d, 'max_temp': t})
        
        # Case B: Data received as a standard List (Row-oriented)
        # Included in case API structure changes
        elif isinstance(forecast_data, list):
            for item in forecast_data:
                daily_forecasts.append({
                    'date': item.get('ForecastDate'),
                    'max_temp': item.get('MaximumTemperature')
                })

        # ========================================================
        
        if not daily_forecasts: continue

        # --- Prepare map data (using tomorrow's data = index 0) ---
        tomorrow = daily_forecasts[0]
        try:
            t_val = float(tomorrow['max_temp'])
        except:
            t_val = 0
        map_data.append({'province': prov_name, 'max_temp': t_val})

        # --- Prepare graph data (Target province only) ---
        if prov_name == target_province:
            for day in daily_forecasts:
                try:
                    g_temp = float(day['max_temp'])
                    graph_dates.append(day['date'])
                    graph_temps.append(g_temp)
                except:
                    continue

    # -------------------------------------------------------
    # Plotting Section
    # -------------------------------------------------------
    # 1. Line Graph
    if graph_dates:
        plt.figure(figsize=(10, 5))
        plt.plot(graph_dates, graph_temps, marker='o', color='orange')
        plt.title(f"7-Day Forecast: {target_province}")
        plt.ylabel("Temp (C)")
        plt.grid()
        plt.show()
        print(f"Graph generated successfully!")
    else:
        print(f"Graph data not found (Please check 'target_province' name)")

    # 2. Map
    try:
        th_geo_url = "https://raw.githubusercontent.com/apisit/thailand.json/master/thailand.json"
        gdf = gpd.read_file(th_geo_url)
        df = pd.DataFrame(map_data)
        
        if not df.empty:
            merged = gdf.merge(df, left_on='name', right_on='province', how='left')
            fig, ax = plt.subplots(1, 1, figsize=(8, 12))
            merged.plot(column='max_temp', ax=ax, legend=True, cmap='coolwarm',
                        missing_kwds={'color': 'lightgrey'})
            plt.title("Thailand Forecast Map")
            plt.show()
            print("Map generated successfully!")
    except Exception as e:
        print(f"Map Error: {e}")

if __name__ == "__main__":
    process_forecast_7days()
------
