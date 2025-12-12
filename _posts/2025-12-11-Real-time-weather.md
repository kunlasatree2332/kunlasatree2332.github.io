---
title: 'Real-time Weather Station Analysis and Visualization'
date: 2025-12-11
permalink: /posts/real-time-station-mapping
tags:
  - Geospatial Mapping
---

Short-term weather monitoring relies on granular station data. However, specific forecast keys within API responses (such as Forecast3Hours versus WeatherForecast) may vary or be missing entirely. A robust script must implement dynamic key searching and fallback mechanisms—switching to raw "Observation" data when specific forecasts are unavailable—to ensure continuous data availability.

This approach allows us to visualize multiple weather variables simultaneously on a geospatial plot. By mapping station coordinates, we can generate a complex bubble map where color represents temperature and bubble size indicates rainfall intensity, providing an immediate visual summary of storm cells and heat pockets across the country.

<img width="1189" height="590" alt="image" src="https://github.com/user-attachments/assets/d3a2d2cd-f160-42a2-91cb-ad79655cf7e8" />

<img width="820" height="1276" alt="image" src="https://github.com/user-attachments/assets/3374225c-ccf1-4106-ac4a-133daf3fbc0c" />


---python
import requests
import pandas as pd
import matplotlib.pyplot as plt
import geopandas as gpd
import seaborn as sns
import urllib3
import numpy as np

# Visualization style
sns.set_style("whitegrid")
urllib3.disable_warnings()


# ==========================================
# 1. API Configuration
# ==========================================
UID = "api"          # Your UID
UKEY = "api12345"    # Your UKEY
BASE_URL = "http://data.tmd.go.th/api"


def get_tmd_data(endpoint):
    url = f"{BASE_URL}/{endpoint}"
    params = {'uid': UID, 'ukey': UKEY, 'format': 'json'}

    try:
        response = requests.get(url, params=params, timeout=30, verify=False)
        return response.json()
    except Exception as e:
        print(f"Connection Error: {e}")
        return None


# ==========================================
# 2. Main Function (API No.4 - Weather3Hours)
# ==========================================
def process_weather_3hours_visuals():

    print("\n" + "="*50)
    print("Retrieving 3-Hour Weather Forecast (API No.4)")
    print("="*50)

    # API Endpoint
    data = get_tmd_data("Weather3Hours/v2")
    if not data:
        return

    # Extract structure
    root = data.get('Weather3Hours', data)
    stations_container = root.get('Stations', {})
    station_list = stations_container.get('Station', [])

    if isinstance(station_list, dict):
        station_list = [station_list]

    print(f"Stations found: {len(station_list)}")

    rows = []

    # Find forecast key dynamically
    target_key = ""
    possible_keys = ['WeatherForecast', 'Forecast3Hours', 'Forecasts', 'WeatherForecasts']

    if station_list:
        first_keys = list(station_list[0].keys())

        # Check expected forecasting keys
        for k in possible_keys:
            if k in first_keys:
                target_key = k
                break

        # If still not found, match substring "Forecast"
        if not target_key:
            for k in first_keys:
                if 'Forecast' in k:
                    target_key = k
                    break

    print(f"Forecast key detected: {target_key}")

    # Extract data
    for st in station_list:

        name = st.get('StationNameEnglish', '').strip()
        lat = st.get('Latitude', 0)
        lon = st.get('Longitude', 0)

        # Extract forecasts
        forecasts = st.get(target_key, [])
        if isinstance(forecasts, dict):
            forecasts = [forecasts]

        # If no forecast data, fall back to observation data
        if not forecasts:
            forecasts = st.get('Observation', [])
            if isinstance(forecasts, dict):
                forecasts = [forecasts]

        if forecasts:
            latest = forecasts[0]

            # Temperature fallback search
            temp = latest.get(
                'AirTemperature',
                latest.get('Temperature', latest.get('MaximumTemperature', 0))
            )

            # Rainfall fallback search
            rain = latest.get(
                'Rainfall',
                latest.get('Rainfall24Hour', latest.get('RainfallPercentage', 0))
            )

            try:
                rows.append({
                    'station': name,
                    'lat': float(lat),
                    'lon': float(lon),
                    'temp': float(temp),
                    'rain': float(rain)
                })
            except:
                continue

    if not rows:
        print("No valid numeric data found for visualization.")
        return

    df = pd.DataFrame(rows)
    print(f"Data processed successfully: {len(df)} stations")

    # ========================================================
    # 1. BAR CHART - Top 15 Highest Temperatures
    # ========================================================

    top_hot = df.sort_values('temp', ascending=False).head(15)

    plt.figure(figsize=(12, 6))
    norm = plt.Normalize(df['temp'].min(), df['temp'].max())
    colors = plt.cm.YlOrRd(norm(top_hot['temp']))

    bars = plt.bar(top_hot['station'], top_hot['temp'],
                   color=colors, edgecolor='black', alpha=0.9)

    for bar in bars:
        height = bar.get_height()
        plt.text(bar.get_x() + bar.get_width()/2., 
                 height + 0.1, f'{height:.1f}°C',
                 ha='center', va='bottom', fontsize=9, fontweight='bold')

    plt.title("Top 15 Hottest Stations (Next 3 Hours Forecast)", fontsize=16, fontweight='bold')
    plt.ylabel("Temperature (°C)", fontsize=12)
    plt.xticks(rotation=45, ha='right')
    plt.grid(axis='y', linestyle='--', alpha=0.5)
    plt.tight_layout()
    plt.show()

    print("Bar chart generated successfully.")

    # ========================================================
    # 2. MAP - Bubble Map (Temperature color, Rain size)
    # ========================================================
    try:
        gdf = gpd.read_file(
            "https://raw.githubusercontent.com/apisit/thailand.json/master/thailand.json"
        )

        fig, ax = plt.subplots(1, 1, figsize=(10, 16))

        gdf.plot(ax=ax, color='#fdfcf0', edgecolor='darkgrey', linewidth=0.8)

        # Bubble size based on rainfall
        sizes = df['rain'].apply(lambda x: x*20 + 30 if x > 0 else 30)

        scatter = ax.scatter(
            df['lon'], df['lat'],
            c=df['temp'],
            s=sizes,
            cmap='coolwarm',
            edgecolor='black',
            linewidth=0.5,
            alpha=0.8
        )

        cbar = plt.colorbar(scatter, ax=ax, fraction=0.03, pad=0.04)
        cbar.set_label("Temperature (°C)", fontsize=12)

        plt.title("3-Hour Forecast Map: Temperature (Color) and Rainfall (Bubble Size)",
                  fontsize=16, fontweight='bold')

        plt.axis('off')

        legend_text = (
            "Legend:\n"
            "- Color: Red = Hotter, Blue = Cooler\n"
            "- Bubble Size: Larger circles indicate higher rainfall"
        )

        props = dict(boxstyle='round', facecolor='white', alpha=0.8)

        ax.text(0.02, 0.02, legend_text, transform=ax.transAxes,
                fontsize=10, verticalalignment='bottom', bbox=props)

        plt.show()
        print("Map generated successfully.")

    except Exception as e:
        print(f"Map Error: {e}")


if __name__ == "__main__":
    process_weather_3hours_visuals()

------
