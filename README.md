# GPS Logger for Raspberry Pi

This project sets up a GPS logger on a Raspberry Pi that records GPS data every 2 meters of movement and pauses logging when stationary. The data is saved in GPX format and stored in a specific directory. This README provides instructions on how to set up, configure, and run the GPS logger.

I used ChatGPS with this initial prompt, and many error reiterations to finish the job:
Explain how to setup a RPI3 and USB G-Mouse to create a GPS logger. The Logger must save a file each new day (Sydney, Australia time) in YYYYMMDD-GPS.gpx format.
The log file should include: Lat,Long,elevation,speed,date,time, satellites.

When I finally got it working I even asked ChatGPT to write the stuff I needed for Github:

## Requirements

- Raspberry Pi 3 or later
- GPS receiver (e.g., USB G-Mouse)
- Raspbian OS or similar Debian-based system
- Python 3
- Virtual environment (optional but recommended)

## Setup Instructions

### 1. Prepare Your Raspberry Pi

1.a. **Update and Upgrade System Packages**:
   ```bash
   sudo apt update
   sudo apt upgrade

1.b  **Install Required Packages**:
Install Required Packages:
bash
Copy code
sudo apt install python3-pip python3-venv gpsd gpsd-clients python3-gps
2. Connect and Configure the GPS Receiver
Connect the GPS Receiver to the Raspberry Pi via USB.

Determine the Device Path:

bash
Copy code
dmesg | grep tty
Look for a device like /dev/ttyACM0.

Configure gpsd:
Edit the configuration file:

bash
Copy code
sudo nano /etc/default/gpsd
Ensure it contains:

bash
Copy code
START_DAEMON="true"
GPSD_OPTIONS="-n"
DEVICES="/dev/ttyACM0"  # Replace with your GPS device path
USBAUTO="true"
Restart the gpsd service:

bash
Copy code
sudo systemctl restart gpsd
3. Create and Activate a Virtual Environment
Create a Virtual Environment:

bash
Copy code
python3 -m venv myenv
Activate the Virtual Environment:

bash
Copy code
source myenv/bin/activate
Install Required Python Packages:

bash
Copy code
pip install gps3 gpxpy geopy
4. Create the Python Script
Save the following Python script as gps_logger.py:

python
Copy code
from gps3 import gps3
import gpxpy
import gpxpy.gpx
import time
from datetime import datetime
import os
from geopy.distance import great_circle

# Initialize GPS socket and data stream
gps_socket = gps3.GPSDSocket()
gps_stream = gps3.DataStream()

# Connect to GPSD
gps_socket.connect()
gps_socket.watch()

# Define constants
LOG_INTERVAL_METERS = 2  # Log every 2 meters
STATIONARY_THRESHOLD_SECONDS = 60  # Time to consider as stationary before pausing

def create_gpx_file():
    now = datetime.now()
    filename = now.strftime("%Y%m%d") + "-GPS.gpx"
    directory = "/home/gps/GPS Logs"  # Directory where GPX files will be saved
    if not os.path.exists(directory):
        os.makedirs(directory)  # Create the directory if it does not exist
    filepath = os.path.join(directory, filename)
    gpx = gpxpy.gpx.GPX()
    return filepath, gpx

def log_gps_data(filepath, gpx):
    last_position = None
    last_log_time = datetime.now()
    stationary_start_time = datetime.now()
    logging = True

    while True:
        for new_data in gps_socket:
            if new_data:
                gps_stream.unpack(new_data)

                # Extract GPS data
                latitude = gps_stream.TPV.get('lat')
                longitude = gps_stream.TPV.get('lon')
                elevation = gps_stream.TPV.get('alt', 0)  # Default to 0 if altitude is not available
                speed = gps_stream.TPV.get('speed', 0)    # Default to 0 if speed is not available
                date = gps_stream.TPV.get('time', 'n/a')  # Default to 'n/a' if time is not available

                if latitude is not None and longitude is not None and date != 'n/a':
                    current_time = datetime.utcnow()

                    # Convert date to datetime object
                    try:
                        time_obj = datetime.strptime(date, "%Y-%m-%dT%H:%M:%S")
                    except ValueError:
                        time_obj = current_time

                    # Check if we need to log data
                    if last_position:
                        distance_moved = great_circle((last_position[0], last_position[1]), (latitude, longitude)).meters
                        if distance_moved >= LOG_INTERVAL_METERS:
                            # Log data
                            gpx.waypoints.append(gpxpy.gpx.GPXWaypoint(
                                latitude=latitude,
                                longitude=longitude,
                                elevation=elevation,
                                time=time_obj
                            ))

                            # Save to file
                            with open(filepath, 'w') as f:
                                f.write(gpx.to_xml())

                            # Update last position and time
                            last_position = (latitude, longitude)
                            last_log_time = current_time
                            stationary_start_time = current_time

                            print(f"Logged data at {latitude}, {longitude}.")

                    else:
                        # Initial position
                        last_position = (latitude, longitude)
                        last_log_time = current_time

                    # Check if the device is stationary
                    if speed == 0:
                        # Check if we've been stationary for too long
                        if (current_time - stationary_start_time).total_seconds() >= STATIONARY_THRESHOLD_SECONDS:
                            logging = False
                            print("Device is stationary. Pausing logging.")
                    else:
                        # Device is moving, reset stationary timer and resume logging
                        stationary_start_time = current_time
                        logging = True

                    if logging:
                        time.sleep(10)  # Small sleep to avoid excessive CPU usage
                    else:
                        time.sleep(60)  # Sleep longer when stationary

if __name__ == "__main__":
    filepath, gpx = create_gpx_file()
    log_gps_data(filepath, gpx)
5. Run the Script
Ensure Virtual Environment is Activated:

bash
Copy code
source myenv/bin/activate
Run the Script:

bash
Copy code
python3 gps_logger.py
Verify Output:

Check the directory /home/gps/GPS Logs to ensure GPX files are being created and updated as expected.

Troubleshooting
gpsd Service Issues:

Ensure gpsd is properly configured in /etc/default/gpsd.
Restart the service with sudo systemctl restart gpsd.
Check status with systemctl status gpsd.
Python Package Errors:

Ensure all required packages are installed in the virtual environment.
Use pip to install missing packages.
GPS Data Issues:

Verify that the GPS receiver is properly connected and recognized.
Check the GPS data using cgps or gpsmon.
Contributing
Feel free to contribute by opening issues, suggesting improvements, or submitting pull requests.

License
This project is licensed under the MIT License. See the LICENSE file for details.

Acknowledgments
Thanks to the GPSD project for providing the GPS daemon and the libraries used in this project.

