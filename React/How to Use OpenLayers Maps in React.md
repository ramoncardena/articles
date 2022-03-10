
# How to Use OpenLayers Maps in React

Let us map it out…

![Screenshot by the author](https://cdn-images-1.medium.com/max/6720/1*1XFvsDgsLWnAL1GFnaDTOw.png)*Screenshot by the author*

Recently I had the task of implementing OpenLayers into our React application at my work. I found an existing npm package that somewhat did how I wanted to use it for maps, but not 100%. Also, the package didn’t have a ton of downloads and didn’t see that the project was being actively supported.

So considering all those factors I decided to build my own OpenLayers map components so that I could have the flexibility I wanted right out of the gate. This is the classic build your own vs. use something that already exists problems that developers face daily.

While I tend to lean heavily towards “don’t reinvent the wheel” normally, there are situations where you know you are going to bump into the wall as far as flexibility goes if you try to shoehorn your use case into something already existing.

So all that being said I will walk you through the components I built to make this work exactly how I wanted it to work and also leaving plenty of flexibility to add on to it in the future.

I will make all the example code available on [Github](https://github.com/mbrown3321/openlayers-react-map).

## Time for the code…

OpenLayers provides an awesome and powerful API via its npm package. this works great, but I wanted to build React components around the API. We will start going through the example code now.

The first step would be adding the OpenLayers npm package to your project.

    npm install ol --save

This should add the package to your dependencies in the package.json file of your project.

Now that we have the OpenLayers API pulled in we can start building the components. We will start with the map component that is the main container for a map instance in the application.

***/src/Map/MapContext.js***

    import React from "react";

    const MapContext = new React.createContext();

    export default MapContext;

***/src/Map/Map.css***

    .ol-map {

      min-width: 600px;

      min-height: 500px;

      margin: 50px;

      height: 500px;

      width: "100%";

    }

    .ol-control {

      position: absolute;

      background-color: rgba(255,255,255,0.4);

      border-radius: 4px;

      padding: 2px;

    }

    .ol-full-screen {

      top: .5em;

      right: .5em;

    }

***/src/Map/Map.js***

    import React, { useRef, useState, useEffect } from "react"

    import "./Map.css";

    import MapContext from "./MapContext";

    import * as ol from "ol";

    
    const Map = ({ children, zoom, center }) => {

      const mapRef = useRef();

      const [map, setMap] = useState(null);

      // on component mount

      useEffect(() => {

        let options = {

          view: new ol.View({ zoom, center }),

          layers: [],

          controls: [],

          overlays: []

        };

        let mapObject = new ol.Map(options);

        mapObject.setTarget(mapRef.current);

        setMap(mapObject);

        return () => mapObject.setTarget(undefined);

      }, []);

      // zoom change handler

      useEffect(() => {

        if (!map) return;

        map.getView().setZoom(zoom);

      }, [zoom]);

      // center change handler

      useEffect(() => {

        if (!map) return;

        map.getView().setCenter(center)

      }, [center])

      return (

        <MapContext.Provider value={{ map }}>

          <div ref={mapRef} className="ol-map">

            {children}

          </div>

        </MapContext.Provider>

      )

    }

    export default Map;

A few things to call out here:

* We are creating a context to pass the *map *object once it is initialized. This object will be where most of the interactions with the OpenLayers API take place.

* When the component mounts we have a *useEffect *which will initialize the map object and store it as the current state. At the end of this, we return a function that sets the target of the map to undefined which disposes of a map once the component unmounts.

* We have created a couple more *useEffect* watchers to detect changes in the *center* and *zoom* props. That way if you want to change the view based on an interaction you can do so by changing the props you are passing.

## Layers…

Now that we have the container for our map we will need to be able to display layers on the map. There are several different layer types provided by OpenLayers, but for this example, we will be using a *TileLayer* and a *VectorLayer.*

We will use the *TileLayer* to display an OpenStreetMap as the base layer and will be using VectorLayer to display some GeoJSON polygons. So first we need to build the components.

***/src/Layers/Layers.js***

    import React from "react";

    const Layers = ({ children }) => {

      return <div>{children}</div>;

    };

    export default Layers;

***/src/Layers/TileLayer.js***

    import { useContext, useEffect } from "react";

    import MapContext from "../Map/MapContext";

    import OLTileLayer from "ol/layer/Tile";

    const TileLayer = ({ source, zIndex = 0 }) => {

      const { map } = useContext(MapContext); 

      useEffect(() => {

        if (!map) return;

        
        let tileLayer = new OLTileLayer({

          source,

          zIndex,

        });

        map.addLayer(tileLayer);

        tileLayer.setZIndex(zIndex);

        return () => {

          if (map) {

            map.removeLayer(tileLayer);

          }

        };

      }, [map]);

      return null;

    };

    export default TileLayer;

***/src/Layers/VectorLayer.js***

    import { useContext, useEffect } from "react";

    import MapContext from "../Map/MapContext";

    import OLVectorLayer from "ol/layer/Vector";

    const VectorLayer = ({ source, style, zIndex = 0 }) => {

      const { map } = useContext(MapContext);

      useEffect(() => {

        if (!map) return;

        let vectorLayer = new OLVectorLayer({

          source,

          style

        });

        map.addLayer(vectorLayer);

        vectorLayer.setZIndex(zIndex);

        return () => {

          if (map) {

            map.removeLayer(vectorLayer);

          }

        };

      }, [map]);

      return null;

    };

    export default VectorLayer;

The *Layers* component is just a placeholder to put all of our layers in and we will look at an example of how that looks at the end of this article.

The TileLayer and VectorLayer components are very similar as you can see. Both use *useEffect *to initialize the layer and call *addLayer *on the *map *object from the context to add themselves to the map.

And just like with the map the layers remove themselves from the map on unmount. There are many more options you can pass for initialization in the OpenLayers API if you want to add those to the props, but for simplicity’s sake, we just have a couple of props we are accepting in the example.

## Controls…

OpenLayers provides several default controls and the ability to make your own custom controls. For our example, I will be just adding a simple full-screen control for the map so you can switch it into full-screen mode.

We will need a container placeholder for the controls just we did for the layers.

***/src/Controls/Controls.js***

    import React from "react";

    const Controls = ({ children }) => {

      return <div>{children}</div>;

    };

    export default Controls;

***/src/Controls/FullScreenControl.js***

    import React, { useContext, useEffect, useState } from "react";

    import { FullScreen } from "ol/control";

    import MapContext from "../Map/MapContext";

    const FullScreenControl = () => {

      const { map } = useContext(MapContext);

      useEffect(() => {

        if (!map) return;

        let fullScreenControl = new FullScreen({});

        map.controls.push(fullScreenControl);

        
        return () => map.controls.remove(fullScreenControl);

      }, [map]);
    

      return null;

    };

    export default FullScreenControl;

Pretty straight forward here. Very similar to how the layer components worked. We initialize the control and then add it to the map on the component mount.

## Putting it all together…

We have built all the map components we need to put together a basic working example so let’s see an example of an actual map.

The only other code in the example project that I haven’t put in the article is I created some wrappers for the OpenLayers source functions. Probably not necessary, but handy for adding default parameters if you don’t want to specify those every time. You can see those examples in the Github project.

So here is what a fully working example map would look like:

***/src/App.js***

    import React, { useState } from 'react';

    import './App.css';

    import Map from "./Map";

    import { Layers, TileLayer, VectorLayer } from "./Layers";

    import { Circle as CircleStyle, Fill, Stroke, Style } from 'ol/style';

    import { osm, vector } from "./Source";

    import { fromLonLat, get } from 'ol/proj';

    import GeoJSON from 'ol/format/GeoJSON';

    import { Controls, FullScreenControl } from "./Controls";

    let styles = {

      'MultiPolygon': new Style({

        stroke: new Stroke({

          color: 'blue',

          width: 1,

        }),

        fill: new Fill({

          color: 'rgba(0, 0, 255, 0.1)',

        }),

      }),

    };

    const geojsonObject = { ... }; // see full geojson object in Github

    const geojsonObject2 = { ... }; // see full geojson object in Github

    const App = () => {

      const [center, setCenter] = useState([-94.9065, 38.9884]);

      const [zoom, setZoom] = useState(9);

      const [showLayer1, setShowLayer1] = useState(true);

      const [showLayer2, setShowLayer2] = useState(true);

    return (

      <div>

        <Map center={fromLonLat(center)} zoom={zoom}>

          <Layers>

            <TileLayer

              source={osm()}

              zIndex={0}

            />

            {showLayer1 && (

              <VectorLayer

                source={vector({ features: new GeoJSON().readFeatures(geojsonObject, { featureProjection: get('EPSG:3857') }) })}

                style={styles.MultiPolygon}

              />

            )}

            {showLayer2 && (

              <VectorLayer

                source={vector({ features: new          GeoJSON().readFeatures(geojsonObject2, { featureProjection:               get('EPSG:3857') }) })}

                style={styles.MultiPolygon}

              />

            )}

          </Layers>

          <Controls>

            <FullScreenControl />

          </Controls>

        </Map>

        <div>

          <input

            type="checkbox"

            checked={showLayer1}

            onChange={event => setShowLayer1(event.target.checked)}

          /> Johnson County

        </div>

        <div>

          <input

            type="checkbox"

            checked={showLayer2}

            onChange={event => setShowLayer2(event.target.checked)}

          /> Wyandotte County</div>

        </div>

      );

    }

    export default App;

This is what a full example would look like. We have a base map *TileLayer *with an OpenStreetMap map to display our vectors on top of.

The two vector layers are sourced from two GeoJSON Polygons in the example. We give it a basic style and it gets rendered on the map. Awesome right?

The other part of the demo is there are checkboxes for interacting with the layers. You can see you can add and remove the layers by checking and unchecking the corresponding boxes.

![Screenshot by the author](https://cdn-images-1.medium.com/max/3708/1*oE_ymaEG_rrQ5DdW7NgbWA.png)*Screenshot by the author*

## Wrap Up…

I hope this demonstration is useful if you find yourself in a spot like I did where you need to use OpenLayers in a React project. Feel free to [browse the code](https://github.com/mbrown3321/openlayers-react-map) and let me know if you have any questions. Thanks for reading!
