# Berlin Interactive Map

This project creates an interactive map of Berlin, showcasing various points of interest including restaurants, hotels, museums, beaches, and airports. The map is built using Mapbox GL JS and displays data from custom GeoJSON files.

## Project Overview

1. Data Preparation:
   - Excel files containing addresses of various locations in Berlin were processed using a Python script.
   - The script utilized Google Maps Geocoding API to obtain latitude and longitude coordinates for each address.
   - GeoJSON files were then generated from this geocoded data.

2. Web Application:
   - An interactive web map was created using Mapbox GL JS.
   - The map displays different categories of locations (restaurants, hotels, museums, beaches, airports) as custom icons.
   - Users can toggle visibility of different categories and get directions to selected locations.

## Features

- Interactive map of Berlin
- Custom icons for different categories of locations
- Category toggle functionality
- Directions service using Mapbox Directions API
- Popup information for each location

## Technologies Used

- HTML/CSS/JavaScript
- Mapbox GL JS
- Mapbox Directions API
- Python (for data processing)
- Google Maps Geocoding API

## Data Processing

The Python script used for geocoding and creating GeoJSON files:

```python
import pandas as pd
from geopy.geocoders import Nominatim
from geopy.exc import GeocoderTimedOut, GeocoderServiceError
import geojson
import time
import os
from google.colab import files

def geocode_address(address):
    if pd.isna(address) or address == 'Adresse':
        return None
    geolocator = Nominatim(user_agent="berlin_poi_app", timeout=10)
    try:
        location = geolocator.geocode(address + ", Berlin, Germany")
        if location:
            return location.latitude, location.longitude
        else:
            print(f"Could not geocode address: {address}")
            return None
    except (GeocoderTimedOut, GeocoderServiceError):
        print(f"Geocoding service timed out for address: {address}. Retrying...")
        time.sleep(2)
        return geocode_address(address)

def excel_to_geojson(excel_file, category):
    # Read Excel file
    df = pd.read_excel(excel_file, header=None)
    
    print(f"Columns in {category} file: {df.columns.tolist()}")
    
    # Determine column names based on category
    if category == 'airports':
        df.columns = ['address', 'name']
        address_col = 'address'
        name_col = 'name'
    elif category == 'beaches':
        df.columns = ['address', 'name']
        address_col = 'address'
        name_col = 'name'
    elif category == 'hotels':
        df = df.iloc[5:]  # Skip the first 5 rows
        df.columns = ['name', 'address', 'phone', 'email'] + [f'col_{i}' for i in range(5, len(df.columns))]
        address_col = 'address'
        name_col = 'name'
    elif category == 'restaurants':
        df.columns = ['address', 'name', 'phone']
        address_col = 'address'
        name_col = 'name'
    elif category == 'museums':
        df.columns = ['address', 'name']
        address_col = 'address'
        name_col = 'name'
    
    print(f"Using '{address_col}' for address and '{name_col}' for name")
    
    # Geocode addresses
    df['coordinates'] = df[address_col].apply(geocode_address)
    
    # Create GeoJSON features
    features = []
    for _, row in df.iterrows():
        if row['coordinates']:
            properties = {
                'name': str(row[name_col]),
                'address': str(row[address_col]),
                'category': category,
                'icon': f"{category}-icon"
            }
            # Add all other columns to properties
            for col in df.columns:
                if col not in [address_col, name_col, 'coordinates']:
                    properties[col] = str(row[col])
            
            feature = geojson.Feature(
                geometry=geojson.Point((row['coordinates'][1], row['coordinates'][0])),
                properties=properties
            )
            features.append(feature)
    
    # Create GeoJSON feature collection
    feature_collection = geojson.FeatureCollection(features)
    
    # Save GeoJSON file
    output_filename = f"{category}.geojson"
    with open(output_filename, 'w') as f:
        geojson.dump(feature_collection, f)
    
    print(f"Created GeoJSON file: {output_filename}")
    files.download(output_filename)

# List of Excel files and their corresponding categories
excel_files = [
    ('/content/Airport in Berlin (1).xlsx', 'airports'),
    ('/content/Beaches in Berlin.xlsx', 'beaches'),
    ('/content/Bestandsliste Hotels in Berlin.xlsx', 'hotels'),
    ('/content/Restaurant in Berlin.xlsx', 'restaurants'),
    ('/content/museum in Berlin.xlsx', 'museums')
]

# Process each Excel file
for excel_file, category in excel_files:
    if os.path.exists(excel_file):
        print(f"\nProcessing {excel_file}...")
        try:
            excel_to_geojson(excel_file, category)
        except Exception as e:
            print(f"Error processing {excel_file}: {str(e)}")
    else:
        print(f"File not found: {excel_file}")

print("All files processed.")
