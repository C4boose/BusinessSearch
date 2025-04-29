import tkinter as tk
from tkinter import messagebox, simpledialog, filedialog
from tkinter import ttk
import requests
import pandas as pd
import time
import json
import os
import threading

CONFIG_FILE = "config.json"
stop_flag = False

# --- API Keys ---
def load_api_keys():
    if os.path.exists(CONFIG_FILE):
        with open(CONFIG_FILE, "r") as f:
            config = json.load(f)
            return config.get("google_api_key")
    else:
        google_key = simpledialog.askstring("API Key", "Enter your Google Places API Key:")
        if google_key:
            with open(CONFIG_FILE, "w") as f:
                json.dump({"google_api_key": google_key}, f)
            return google_key
        else:
            messagebox.showerror("Missing Key", "Google API key is required to run the app.")
            exit()

def change_api_keys():
    google_key = simpledialog.askstring("API Key", "Enter new Google Places API Key:")
    if google_key:
        with open(CONFIG_FILE, "w") as f:
            json.dump({"google_api_key": google_key}, f)
        global GOOGLE_API_KEY
        GOOGLE_API_KEY = google_key
        messagebox.showinfo("API Key Updated", "API key has been successfully updated.")

# --- Google Places Search ---
def search_businesses_google(api_key, location, keyword, radius_meters, max_results=200):
    url = "https://maps.googleapis.com/maps/api/place/nearbysearch/json"
    params = {
        "key": api_key,
        "location": location,
        "radius": radius_meters,
    }

    if keyword and keyword != "Any":
        special_keywords = {
            "House Removals": "house removals",
            "Waste Removals": "waste removals",
            "Van Rentals": "van rentals"
        }
        params["keyword"] = special_keywords.get(keyword, keyword)
    else:
        params["keyword"] = "business"

    all_results = []
    total_results = 0

    while total_results < max_results:
        # Send the request for this page
        response = requests.get(url, params=params)
        if response.status_code != 200:
            raise Exception(f"Google Places API Error: {response.status_code} - {response.text}")

        # Get the results for this page
        results = response.json().get("results", [])
        all_results.extend(results)
        total_results += len(results)

        # Update the status
        status_label.config(text=f"Found {total_results} businesses...")

        # If we have enough results, stop early
        if total_results >= max_results:
            break

        # Check if there is a next page
        next_page_token = response.json().get("next_page_token")
        if not next_page_token:
            break

        # Wait for 2 seconds before requesting the next page (required by Google)
        time.sleep(2)

        # Update parameters to request the next page
        params = {
            "key": api_key,
            "pagetoken": next_page_token
        }

    return all_results

def search_businesses_google_with_grid(api_key, center_location, keyword, radius_meters, max_results=200):
    import math

    def calculate_new_coordinates(lat, lng, dx, dy):
        """Calculate new latitude and longitude based on distance offsets (dx, dy) in meters."""
        earth_radius = 6378137  # Earth's radius in meters
        new_lat = lat + (dy / earth_radius) * (180 / math.pi)
        new_lng = lng + (dx / (earth_radius * math.cos(math.pi * lat / 180))) * (180 / math.pi)
        return new_lat, new_lng

    # Parse the center location into latitude and longitude
    center_lat, center_lng = map(float, center_location.split(','))

    # Define the grid size (e.g., 2 * radius for each cell to avoid overlap)
    grid_size = radius_meters * 2

    # Create a list to store all results
    all_results = []

    # Define the number of grid cells in each direction (adjust as needed)
    num_cells = math.ceil(math.sqrt(max_results / 60))  # Approximation based on 60 results per query

    for i in range(-num_cells, num_cells + 1):
        for j in range(-num_cells, num_cells + 1):
            # Calculate the center of the current grid cell
            cell_lat, cell_lng = calculate_new_coordinates(center_lat, center_lng, i * grid_size, j * grid_size)

            # Perform a search for the current grid cell
            url = "https://maps.googleapis.com/maps/api/place/nearbysearch/json"
            params = {
                "key": api_key,
                "location": f"{cell_lat},{cell_lng}",
                "radius": radius_meters,
                "keyword": keyword if keyword != "Any" else "business"
            }

            while True:
                response = requests.get(url, params=params)
                if response.status_code != 200:
                    raise Exception(f"Google Places API Error: {response.status_code} - {response.text}")

                results = response.json().get("results", [])
                all_results.extend(results)

                # Check if we have enough results
                if len(all_results) >= max_results:
                    return all_results[:max_results]

                # Check for the next page token
                next_page_token = response.json().get("next_page_token")
                if not next_page_token:
                    break

                # Wait for 2 seconds before requesting the next page
                time.sleep(2)
                params = {"key": api_key, "pagetoken": next_page_token}

    return all_results[:max_results]

# --- Place Details ---
def get_place_details(api_key, place_id):
    url = "https://maps.googleapis.com/maps/api/place/details/json"
    params = {
        "key": api_key,
        "place_id": place_id,
        "fields": "name,formatted_phone_number,international_phone_number,website"
    }
    response = requests.get(url, params=params)
    if response.status_code != 200:
        raise Exception(f"Place Details API Error: {response.status_code} - {response.text}")
    return response.json().get("result", {})

# --- Geocode ZIP Code to Lat/Lng ---
def geocode_zip(api_key, zip_code):
    url = "https://maps.googleapis.com/maps/api/geocode/json"
    params = {"address": zip_code, "key": api_key}
    response = requests.get(url, params=params)
    if response.status_code != 200:
        raise Exception(f"Geocoding Error: {response.status_code} - {response.text}")
    results = response.json().get("results", [])
    if not results:
        raise Exception("No geocoding results found.")
    location = results[0]['geometry']['location']
    return f"{location['lat']},{location['lng']}"

# --- GUI Logic ---
def stop_search():
    global stop_flag
    stop_flag = True

def run_search_thread():
    global stop_flag
    stop_flag = False

    zip_code = zip_entry.get()
    query = business_type_combobox.get()
    limit = limit_entry.get()
    radius_meters = radius_entry.get()
    file_path = filedialog.asksaveasfilename(defaultextension=".xlsx", filetypes=[("Excel Files", "*.xlsx")])
    run_speed_test = speed_test_var.get()

    if not zip_code:
        messagebox.showerror("Missing Info", "Please enter ZIP/Postal code.")
        return

    if query == "Any":
        query = "business"

    try:
        limit = int(limit) if limit else 50
        radius_meters = int(radius_meters) if radius_meters else 5000  # Default to 5000 meters (5 km)
    except ValueError:
        messagebox.showerror("Invalid Input", "Limit and Radius must be numbers.")
        return

    if not file_path:
        return

    search_button.config(state=tk.DISABLED)
    stop_button.config(state=tk.NORMAL)
    progress_bar.start()
    status_label.config(text="Geocoding ZIP/Postal code...")

    def background_task():
        try:
            latlng = geocode_zip(GOOGLE_API_KEY, zip_code)
            status_label.config(text="Searching for businesses...")

            # Use the grid-based search function
            businesses = search_businesses_google_with_grid(
                GOOGLE_API_KEY, latlng, query, radius_meters, max_results=limit
            )

            results = []
            count = 0

            def fetch_details(b):
                nonlocal count
                if stop_flag or count >= limit:
                    return

                place_id = b.get("place_id")
                details = get_place_details(GOOGLE_API_KEY, place_id)

                name = details.get("name", "N/A")
                phone = details.get("formatted_phone_number", "N/A")
                website = details.get("website")

                if not website:
                    website = f"https://www.google.com/search?q={'+'.join(name.split())}"

                if run_speed_test:
                    website_speed = "Tested"
                else:
                    website_speed = "N/A"

                results.append({"Name": name, "Website": website, "Phone Number": phone, "Speed Test": website_speed})
                count += 1
                status_label.config(text=f"Processed {count}/{limit} businesses...")

            # Use threading to fetch details in parallel
            threads = []
            for b in businesses:
                if stop_flag or count >= limit:
                    break
                thread = threading.Thread(target=fetch_details, args=(b,))
                threads.append(thread)
                thread.start()

            # Wait for all threads to complete
            for thread in threads:
                thread.join()

            # Save results to Excel
            df = pd.DataFrame(results)
            df.to_excel(file_path, index=False)
            messagebox.showinfo("Done", f"Results saved to {file_path}\nSearched {count} businesses.")

        except Exception as e:
            messagebox.showerror("Error", f"Something went wrong: {e}")

        finally:
            search_button.config(state=tk.NORMAL)
            stop_button.config(state=tk.DISABLED)
            progress_bar.stop()
            status_label.config(text="Idle")

    threading.Thread(target=background_task).start()

# --- Load API Keys ---
GOOGLE_API_KEY = load_api_keys()

# --- GUI Setup ---
root = tk.Tk()
root.title("Business Search Tool")
root.geometry("460x600")

# --- Menu Bar ---
menubar = tk.Menu(root)
settings_menu = tk.Menu(menubar, tearoff=0)
settings_menu.add_command(label="Change API Key", command=change_api_keys)
menubar.add_cascade(label="Settings", menu=settings_menu)
root.config(menu=menubar)

# --- GUI Elements ---
tk.Label(root, text="Enter ZIP/Postal Code:").pack(pady=5)
zip_entry = tk.Entry(root, width=40)
zip_entry.pack()

tk.Label(root, text="Select Business Type (or 'Any' for all):").pack(pady=5)
business_type_combobox = ttk.Combobox(
    root,
    values=[
        "Any",
        "House Removals",
        "Waste Removals",
        "Van Rentals"
    ],
    width=40
)
business_type_combobox.set("Any")
business_type_combobox.pack()

tk.Label(root, text="Search Radius (meters, default 5000):").pack(pady=5)
radius_entry = tk.Entry(root, width=40)
radius_entry.pack()

tk.Label(root, text="Max number of businesses to check (default 50):").pack(pady=5)
limit_entry = tk.Entry(root, width=40)
limit_entry.pack()

speed_test_var = tk.BooleanVar()
tk.Checkbutton(root, text="Run Website Speed Test", variable=speed_test_var).pack(pady=5)

search_button = tk.Button(root, text="Search", command=run_search_thread)
search_button.pack(pady=10)

stop_button = tk.Button(root, text="Stop Search", command=stop_search, state=tk.DISABLED)
stop_button.pack(pady=5)

progress_bar = ttk.Progressbar(root, mode='indeterminate', length=300)
progress_bar.pack(pady=5)

status_label = tk.Label(root, text="Idle", anchor="w")
status_label.pack(fill="x", pady=5, padx=10)

root.mainloop()
