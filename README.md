import pandas as pd
import numpy as np

df = pd.read_csv('C:\Users\Win 10\Downloads\dataset.csv')  

# Initialize base price and alpha
BASE_PRICE = 10.0
ALPHA = 2.0
MIN_PRICE = 5.0
MAX_PRICE = 20.0

# Ensure required columns exist
#required_cols = ['Time', 'ParkingLotID', 'Capacity', 'Occupancy']
#assert all(col in df.columns for col in required_cols), "Missing required columns!"

# Sort data by time and parking lot for correct simulation
df = df.sort_values(['ParkingLotID', 'Time'])

# Create a dictionary to store previous prices per lot
previous_prices = {lot_id: BASE_PRICE for lot_id in df['ParkingLotID'].unique()}

# Add a new column for price
prices = []

for _, row in df.iterrows():
    lot_id = row['ParkingLotID']
    occupancy = row['Occupancy']
    capacity = row['Capacity']

    # Avoid division by zero
    if capacity == 0:
        occ_ratio = 0
    else:
        occ_ratio = occupancy / capacity

    # Apply linear pricing update
    prev_price = previous_prices[lot_id]
    new_price = prev_price + ALPHA * occ_ratio

    # Clip to bounds
    new_price = max(MIN_PRICE, min(MAX_PRICE, new_price))

    # Store and update
    prices.append(new_price)
    previous_prices[lot_id] = new_price

# Append price column to the dataframe
df['UpdatedPrice'] = prices

# Preview
df.head()
import matplotlib.pyplot as plt

# Plot price trend for a sample parking lot
sample_lot = df['ParkingLotID'].unique()[0]
sample_data = df[df['ParkingLotID'] == sample_lot]

plt.figure(figsize=(10, 5))
plt.plot(sample_data['Time'], sample_data['UpdatedPrice'], label=f'Lot {sample_lot}')
plt.xlabel('Time')
plt.ylabel('Price ($)')
plt.title('Price Trend for Sample Parking Lot')
plt.legend()
plt.grid(True)
plt.xticks(rotation=45)
plt.tight_layout()
plt.show()
def get_vehicle_weight(vtype):
    return EPSILON.get(vtype.lower(), 1.0)  # Default weight = 1.0

def calculate_demand(row):
    occ_ratio = row['Occupancy'] / row['Capacity'] if row['Capacity'] != 0 else 0
    vehicle_weight = get_vehicle_weight(row['VehicleType'])
    
    demand = (
        ALPHA * occ_ratio +
        BETA * row['QueueLength'] -
        GAMMA * row['Traffic'] +
        DELTA * row['IsSpecialDay'] +
        EPSILON[row['VehicleType']]  # or vehicle_weight
    )
    return demand

# Calculate raw demand
df['DemandRaw'] = df.apply(calculate_demand, axis=1)

# Normalize demand across entire dataset
df['DemandNormalized'] = (df['DemandRaw'] - df['DemandRaw'].min()) / (df['DemandRaw'].max() - df['DemandRaw'].min())

# Apply pricing formula
df['UpdatedPrice'] = BASE_PRICE * (1 + LAMBDA * df['DemandNormalized'])

# Clip price
df['UpdatedPrice'] = df['UpdatedPrice'].clip(MIN_PRICE, MAX_PRICE)
from math import radians, sin, cos, sqrt, atan2

def haversine(lat1, lon1, lat2, lon2):
    R = 6371  # km
    dlat = radians(lat2 - lat1)
    dlon = radians(lon2 - lon1)
    a = sin(dlat / 2)**2 + cos(radians(lat1)) * cos(radians(lat2)) * sin(dlon / 2)**2
    c = 2 * atan2(sqrt(a), sqrt(1 - a))
    return R * c

# Add competitor-adjusted pricing
def adjust_for_competition(row, df, radius_km=0.5):
    lot_id = row['ParkingLotID']
    time = row['Time']
    lat, lon = row['Latitude'], row['Longitude']
    price = row['UpdatedPrice']

    # Nearby lots at same timestamp
    nearby_lots = df[(df['Time'] == time) & (df['ParkingLotID'] != lot_id)]

    competitors = nearby_lots.copy()
    competitors['Distance'] = competitors.apply(
        lambda r: haversine(lat, lon, r['Latitude'], r['Longitude']), axis=1
    )
    nearby = competitors[competitors['Distance'] <= radius_km]

    if not nearby.empty:
        avg_competitor_price = nearby['UpdatedPrice'].mean()

        # Price logic
        if row['Occupancy'] >= row['Capacity'] and avg_competitor_price < price:
            return max(MIN_PRICE, price - 1.0)  # Encourage rerouting
        elif avg_competitor_price > price:
            return min(MAX_PRICE, price + 0.5)  # Increase to match market
    return price

# Apply competitive adjustment
df['FinalPrice'] = df.apply(lambda row: adjust_for_competition(row, df), axis=1)
from bokeh.plotting import figure, show, output_notebook
from bokeh.models import ColumnDataSource
output_notebook()

# Filter data for a single lot
lot_id = df['ParkingLotID'].unique()[0]
lot_df = df[df['ParkingLotID'] == lot_id]

source = ColumnDataSource(lot_df)

p = figure(x_axis_type='auto', title=f'Price Over Time - Parking Lot {lot_id}',
           plot_height=350, plot_width=800)

p.line(x='Time', y='FinalPrice', source=source, line_width=2, color='green', legend_label="Final Price")
p.line(x='Time', y='UpdatedPrice', source=source, line_width=1, color='blue', legend_label="Pre-Competitive Price")

p.xaxis.axis_label = 'Time'
p.yaxis.axis_label = 'Price ($)'
p.legend.location = "top_left"
p.grid.grid_line_alpha = 0.3

show(p)

