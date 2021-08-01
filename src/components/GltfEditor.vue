<template>
  <div class="hello">
    <!-- <input type="file" ref="doc" @change="readFile()" /> -->
  </div>
</template>

<script>

// The URL on your server where CesiumJS's static files are hosted.
window.CESIUM_BASE_URL = "/static/Cesium/";

import * as Cesium from "cesium";
import "cesium/Build/Cesium/Widgets/widgets.css";
Cesium.Ion.defaultAccessToken =
  "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJqdGkiOiI4YmUxM2NkYy01MzFmLTQwMTItOWQ5NS0yYjlhZGYwZjBhMzciLCJpZCI6NTIxMzUsImlhdCI6MTYxODQ1NzQzOH0.2MJvhSVzDUa8tS2AnTTasFMeQbd5neJQeghj3fX2tr4";

import { Document, WebIO } from "@gltf-transform/core";

export default {
  name: "GltfEditor",
  data() {
    return {
      file: null,
      content: null,
    };
  },
  mounted() {
    this.reset();
  },
  methods: {
    createGltf() {
      const doc = new Document();

      const buffer = doc.createBuffer();

      const focal = 10
      const xHalf = 3.5
      const yHalf = 2

      const vertices = new Float32Array([
        0, 0, 0, // origin 0
        -xHalf, -yHalf, focal, // TopL 1
        xHalf, -yHalf, focal, // TopR 2
        -xHalf, yHalf, focal, // BotL 3
        xHalf, yHalf, focal,  // BotR 4
      ])
      const position = doc
        .createAccessor()
        .setType("VEC3")
        .setArray(new Float32Array(vertices))
        .setBuffer(buffer);

      // cone
      const triangles1 = new Uint16Array([
        0, 2, 1,
        0, 1, 2,
        // 0, 1, 3,
        0, 3, 4,
        0, 4, 3,
        // 0, 4, 2,
      ]) 
      const indices1 = doc
        .createAccessor()
        .setType("SCALAR")
        .setArray(new Uint16Array(triangles1))
        .setBuffer(buffer);

      const prim1 = doc
        .createPrimitive()
        .setAttribute("POSITION", position)
        .setIndices(indices1);
      const material1 = doc.createMaterial('myMaterial')
        .setBaseColorFactor([1, 0, 0, 0.5]) // RGBA
      prim1.setMaterial(material1)

      // bottom
      const triangles2 = new Uint16Array([
        1, 2, 3,
        1, 3, 2,
        2, 4, 3,
        2, 3, 4,
      ]) 
      const indices2 = doc
        .createAccessor()
        .setType("SCALAR")
        .setArray(new Uint16Array(triangles2))
        .setBuffer(buffer);

      const prim2 = doc
        .createPrimitive()
        .setAttribute("POSITION", position)
        .setIndices(indices2);
      const material2 = doc.createMaterial('myMaterial')
        .setBaseColorFactor([0, 1, 0, 1]) // RGBA
      prim2.setMaterial(material2)



      const mesh = doc.createMesh();
      mesh.addPrimitive(prim1);
      mesh.addPrimitive(prim2);
      const node = doc.createNode().setMesh(mesh);
      const scene = doc.createScene().addChild(node);

      const glb = new WebIO().writeBinary(doc); // → ArrayBuffer

      console.log(glb)
      return glb
    },
    async reset() {
      var viewer = new Cesium.Viewer("cesiumContainer", {});
      var scene = viewer.scene;
      var context = scene.context;
      var camera = viewer.camera;
      scene.debugShowFramesPerSecond = true;

      var count = 4;
      var spacing = 0.0002;
      var url = "gltf-models/Box.gltf";
      var useCollection = true;

      var centerLongitude = -75.61209431;
      var centerLatitude = 40.042530612;
      var height = 50.0;

      var urlModel = Cesium.Model.fromGltf({ url });
      console.log('urlModel: ', urlModel);

      var transformModel = this.createGltf();
      console.log('transformModel: ', transformModel)

      function orientCamera(center, radius) {
        var range = Math.max(radius, camera.frustum.near) * 2.0;
        var heading = Cesium.Math.toRadians(230.0);
        var pitch = Cesium.Math.toRadians(-20.0);
        camera.lookAt(
          center,
          new Cesium.HeadingPitchRange(heading, pitch, range)
        );
        camera.lookAtTransform(Cesium.Matrix4.IDENTITY);
      }

      function createCollection(instances) {
        var collection = scene.primitives.add(
          new Cesium.ModelInstanceCollection({
            // url: url,
            gltf: transformModel,
            instances: instances,
          })
        );

        collection.readyPromise
          .then(function(collection) {
            // Play and loop all animations at half-speed
            collection.activeAnimations.addAll({
              multiplier: 0.5,
              loop: Cesium.ModelAnimationLoop.REPEAT,
            });
            orientCamera(
              collection._boundingSphere.center,
              collection._boundingSphere.radius
            );
          })
          .otherwise(function(error) {
            window.alert(error);
          });
      }

      function createModels(instances) {
        var points = [];
        var model;

        var length = instances.length;
        for (var i = 0; i < length; ++i) {
          var instance = instances[i];
          var modelMatrix = instance.modelMatrix;
          var translation = new Cesium.Cartesian3();
          Cesium.Matrix4.getTranslation(modelMatrix, translation);
          points.push(translation);

          /////////////////////////////////////////////////////////////////////////////////////////////////////
          /////////////////////////////////////////////////////////////////////////////////////////////////////
          /////////////////////////////////////////////////////////////////////////////////////////////////////
          var dimensions = new Cesium.Cartesian3(400000.0, 300000.0, 500000.0);
          // Box geometries are initially centered on the origin.
          // We can use a model matrix to position the box on the
          // globe surface.
          var positionOnEllipsoid = Cesium.Cartesian3.fromDegrees(-105.0, 45.0);
          // 移到某个位置，抬高25万米
          var boxModelMatrix = Cesium.Matrix4.multiplyByTranslation(
              Cesium.Transforms.eastNorthUpToFixedFrame(positionOnEllipsoid),
              new Cesium.Cartesian3(0.0, 0.0, dimensions.z * 0.5), new Cesium.Matrix4());
          // Create a box geometry.
          var boxGeometry = Cesium.BoxGeometry.fromDimensions({
              vertexFormat : Cesium.PerInstanceColorAppearance.VERTEX_FORMAT,
              dimensions : dimensions
          });
          // Create a geometry instance using the geometry
          // and model matrix created above.
          var boxGeometryInstance = new Cesium.GeometryInstance({
              geometry : boxGeometry,
              modelMatrix : boxModelMatrix,
              attributes : { //指定顶点的颜色
                  color : Cesium.ColorGeometryInstanceAttribute.fromColor(new Cesium.Color(1.0, 0.0, 0.0, 0.5))
              }
          });
          // Add the geometry instance to primitives.
          model = scene.primitives.add(new Cesium.Primitive({
              geometryInstances : boxGeometryInstance,
              appearance : new Cesium.PerInstanceColorAppearance({ //指定材质的颜色 和顶点颜色相同
                  closed: true
              })
          }));


          /////////////////////////////////////////////////////////////////////////////////////////////////////
          /////////////////////////////////////////////////////////////////////////////////////////////////////
          /////////////////////////////////////////////////////////////////////////////////////////////////////
          // model = scene.primitives.add(
          //   Cesium.Model.fromGltf({
          //     url: url,
          //     modelMatrix: modelMatrix,
          //   })
          // );

          model.readyPromise
            .then(function(model) {
              // Play and loop all animations at half-speed
              // model.activeAnimations.addAll({
              //   multiplier: 0.5,
              //   loop: Cesium.ModelAnimationLoop.REPEAT,
              // });
              console.log('create model: ', model)
            })
            .otherwise(function(error) {
              window.alert(error);
            });
        }
      }

      function reset() {
        scene.primitives.removeAll();
        var instances = [];
        var gridSize = Math.sqrt(count);

        for (var y = 0; y < gridSize; ++y) {
          for (var x = 0; x < gridSize; ++x) {
            var longitude = centerLongitude + spacing * (x - gridSize / 2);
            var latitude = centerLatitude + spacing * (y - gridSize / 2);
            var position = Cesium.Cartesian3.fromDegrees(
              longitude,
              latitude,
              height
            );

            var heading = Math.random();
            var pitch = Math.random();
            var roll = Math.random();
            var scale = (Math.random() + 1.0) / 2.0;

            var modelMatrix = Cesium.Transforms.headingPitchRollToFixedFrame(
              position,
              new Cesium.HeadingPitchRoll(heading, pitch, roll)
            );
            Cesium.Matrix4.multiplyByUniformScale(
              modelMatrix,
              scale,
              modelMatrix
            );

            instances.push({
              modelMatrix: modelMatrix,
            });
          }
        }

        if (useCollection) {
          createCollection(instances);
        } else {
          createModels(instances);
        }
      }

      // Scale picked instances
      var handler = new Cesium.ScreenSpaceEventHandler(viewer.canvas);
      handler.setInputAction(function(movement) {
        var pickedInstance = scene.pick(movement.position);
        if (Cesium.defined(pickedInstance)) {
          console.log(pickedInstance);
          var instance = useCollection
            ? pickedInstance
            : pickedInstance.primitive;
          var scaleMatrix = Cesium.Matrix4.fromUniformScale(1.1);
          var modelMatrix = Cesium.Matrix4.multiply(
            instance.modelMatrix,
            scaleMatrix,
            new Cesium.Matrix4()
          );
          instance.modelMatrix = modelMatrix;
        }
      }, Cesium.ScreenSpaceEventType.LEFT_CLICK);

      reset();
    },
  },
};
</script>
