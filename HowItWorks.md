# Introduction #

How it works... or how to get from having your location (latitude, longitude, and altitude) and a Marker's location (latitude, longitude, and altitude) to a floating symbol on the screen.


# Details #

**_(1) The sequence starts with the `SensorsActivty` which the `AugmentedReality` extends._**

## `SensorsActivity` ##
The `SensorsActivity` gets two events:

### `onLocationChanged()` ###
The first is `onLocationChanged(Location location)` which we use to do two things:
  * Set our Location
```
ARData.setCurrentLocation(location)
```
  * Set the `GeomagneticField` and load it into a matrix then rotate to compensate for the phone being in landscape orientation.
```
gmf = new GeomagneticField((float) location.getLatitude(), 
                           (float) location.getLongitude(),
                           (float) location.getAltitude(), 
                           System.currentTimeMillis())

double angleY = Math.toRadians(-gmf.getDeclination());
mageticNorthCompensation.toIdentity();          
mageticNorthCompensation.set(FloatMath.cos(angleY),  0f, FloatMath.sin(angleY), 
                             0f,                     1f, 0f, 
                             -FloatMath.sin(angleY), 0f, FloatMath.cos(angleY));

// GeomagneticField assumes the phone is parallel to the ground with the 
// screen pointing to the sky, you have to rotate it to compensate when 
// working with the screen facing your face in landscape mode
mageticNorthCompensation.prod(xAxisRotation);
```

### `onSensorChanged()` ###
The other event is `onSensorChanged(SensorEvent evt)` which we do a couple of things with:

  * Get the sensor data and smooth it with a `LowPassFilter`
```
if (evt.sensor.getType() == Sensor.TYPE_ACCELEROMETER) {
    smooth = LowPassFilter.filter(evt.values, grav);
    grav[0] = smooth[0];
    grav[1] = smooth[1];
    grav[2] = smooth[2];
} else if (evt.sensor.getType() == Sensor.TYPE_MAGNETIC_FIELD) {
    smooth = LowPassFilter.filter(evt.values, mag);
    mag[0] = smooth[0];
    mag[1] = smooth[1];
    mag[2] = smooth[2];
}
```

  * Get the rotation matrix and remap the coordinates to compensate for landscape orientation.
```
//Get rotation matrix given the gravity and geomagnetic matrices
SensorManager.getRotationMatrix(temp, null, grav, mag);

//Translate the rotation matrices from Y and -X (landscape)
SensorManager.remapCoordinateSystem(temp, SensorManager.AXIS_Y, SensorManager.AXIS_MINUS_X, rotation);
```

  * Get the azimuth, pitch, and roll then convert to degrees and save for use on the radar:
```
//Get the azimuth, pitch, roll
SensorManager.getOrientation(rotation,apr);
float floatAzimuth = (float)Math.toDegrees(apr[0]);
if (floatAzimuth<0) floatAzimuth+=360;  //Convert from -180->180 to 0->359
ARData.setAzimuth(floatAzimuth);
ARData.setPitch((float)Math.toDegrees(apr[1]));
ARData.setRoll((float)Math.toDegrees(apr[2]));
```

  * Save the remapped coordinates into our matrix and save as the rotation matrix:
```
//Convert from float[9] to Matrix
worldCoord.set(rotation[0], rotation[1], rotation[2], 
               rotation[3], rotation[4], rotation[5], 
               rotation[6], rotation[7], rotation[8]);
```

  * Adjust the world coordinate matrix for the difference between true north and magnetic north:
```
//Identity matrix
// [ 1, 0, 0 ]
// [ 0, 1, 0 ]
// [ 0, 0, 1 ]
magneticCompensatedCoord.toIdentity();

//Cross product the matrix with the magnetic north compensation
magneticCompensatedCoord.prod(mageticNorthCompensation);

//Product with the world coordinates, compensate for mag north
magneticCompensatedCoord.prod(worldCoord);

//Invert the matrix, up-down and left-right are reversed when working in landscape mode
magneticCompensatedCoord.invert(); 

//Set the rotation matrix (used to translate all object from lat/lon to x/y/z)
ARData.setRotationMatrix(magneticCompensatedCoord);
```

**_(2) The sequence then goes to the `ARData` class which processes the Markers when the `SensorsActivity` calls the setCurrentLocation() method._**

## `ARData` ##
When the `setCurrentLocation()` method is called, we iterate over the Collection of Markers and update the relative location of each Marker based on the Location passed into the `setCurrentLocation()` method.

### `setCurrentLocation()` ###
```
for(Marker ma: markerList.values()) {
    ma.calcRelativePosition(location);
}
```

**_(3) The sequence continues with the `Marker` class which is called via calcRelativePosition() method._**

  * Convert the latitude, longitude, and altitude of the Marker into a relative x, y, z position. The x, y, and z fields are essentially the distance from the phones location to the `Marker` in meters.

```
PhysicalLocation.convLocationToVector(location, physicalLocation,
                                      locationXyzRelativeToPhysicalLocation);
```

The locationXyzRelativeToPhysicalLocation variable now has the relative x, y, z location of Marker. Where z=latitude distance, x=longitude distance, and y=altitude distance.

_**(4) The sequence then moves to the `update(Canvas canvas, float addX, float addY)` method which is called when the screen is refreshed.**

## ` Marker ` ##_

When the `update()` method is called, we update the camera properties (in case they have changed) and the compensate the x, y, z locations by the azimuth, pitch, and camera view properties.

### `update()` ###
  * Update the camera's view properties.
```
cam.set(canvas.getWidth(), canvas.getHeight(), false);
cam.setViewAngle(CameraModel.DEFAULT_VIEW_ANGLE);
```

### `populateMatrices()` ###
  * Compensate for the azimuth and pitch of the phone. This is done by setting a temp matrix with the relative x, y, z position (locationXyzRelativeToPhysicalLocation) and the take the product of that matrix and the rotation matrix from the `ARData`.
```
tmpSymbolVector.set(symbolVector);
tmpSymbolVector.add(locationXyzRelativeToPhysicalLocation);        
tmpSymbolVector.prod(ARData.getRotationMatrix());
```

  * Compensate for the camera's view properties. Save the result in the symbolXyzRelativeToCameraView variable.
```
cam.projectPoint(tmpSymbolVector, tmpVector, addX, addY);
symbolXyzRelativeToCameraView.set(tmpVector);
```

# Example Narrative #

Phone's location is: `lat=39.98000266 lon=-74.85736476 alt=-9.100000381469727`<br>
A Marker's location is <code>lat=39.95, lon=-74.9, alt=1.0</code> (physicalLocation variable)<br>

In <code>Marker.calcRelativePosition()</code> method....<br>
We use the two locations above as params in <code>PhysicalLocation.convLocationToVector()</code> to figure out the distance in meters on x, y, and z. The Marker's x,y,z in this instance would be <code>x=-3641.8494, y=10.1, z=3331.3142</code> (locationXyzRelativeToPhysicalLocation).<br>

In <code>Marker.populateMatrices()</code> method...<br>
We now get the product between the Marker's x,y,z vector and the rotation matrix (which moves the x,y,z of the Marker relative to azimuth and pitch of the camera. The compensated x, y, z would be <code>x=-755.72363, y=545.7637, z=-4846.8384</code> (tmpSymbolVector).<br>

We then project that vector into a point based on the camera's view which is now <code>x=249.42941, y=131.2619, z=-4846.8384</code> (symbolXyzRelativeToCameraView).<br>

So, the <code>physicalLocation</code> should not change once the Marker is constructed. The <code>locationXyzRelativeToPhysicalLocation</code> variable will change when the phone's location changes. The <code>tmpSymbolVector</code> variable changes based upon the bearing/pitch of the phones. And the <code>symbolXyzRelativeToCameraView</code> changes based on the camera's view (width/height of screen).<br>