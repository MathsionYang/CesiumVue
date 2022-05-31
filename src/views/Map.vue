<template>
  <div id="cesiumContainer">
    <div
      style="
        width: 420px;
        height: 30px;
        position: absolute;
        bottom: 40px;
        right: 200px;
        z-index: 1;
        font-size: 15px;
      "
    >
      <div style="width: 140px; height: 30px; float: left">
        <font size="3" color="white"
          >经度：<span>{{ Position.lat }}</span></font
        >
      </div>
      <div style="width: 140px; height: 30px; float: left">
        <font size="3" color="white"
          >纬度：<span>{{ Position.log }}</span></font
        >
      </div>
      <div style="width: 140px; height: 30px; float: left">
        <font size="3" color="white"
          >视角高：<span>{{ Position.alt }}</span
          >km</font
        >
      </div>
    </div>
  </div>
</template>

<script>
import "cesium/Build/Cesium/Widgets/widgets.css";
import * as Cesium from "cesium";

export default {
  name: "Map",
  data() {
    return {
      Position: {
        lat: "",
        log: "",
        alt: "",
      },
    };
  },
  mounted() {
    this.init();
  },
  methods: {
    init() {
      this.$viewer = new Cesium.Viewer("cesiumContainer", {
        geocoder: false, // 地理位置搜索控件
        homeButton: false, // 首页跳转控件
        sceneModePicker: false, // 2D,3D和Columbus View切换控件
        baseLayerPicker: false, // 三维地图底图切换控件
        navigationHelpButton: false, // 帮助提示控件
        animation: false, // 视图动画播放速度控件
        timeline: false, // 时间轴控件
        fullscreenButton: false, // 全屏控件
        infoBox: false, // 点击显示窗口控件
        selectionIndicator: false, // 实体对象选择框控件
        scene3DOnly: true, // 仅3D渲染，节省GPU内存
      });
      this.$viewer.cesiumWidget.creditContainer.style.display = "none";

      let imageryLayers = this.$viewer.imageryLayers;
      let googleMap = new Cesium.WebMapTileServiceImageryProvider({
        url: "http://t0.tianditu.com/img_w/wmts?service=wmts&request=GetTile&version=1.0.0&LAYER=img&tileMatrixSet=w&TileMatrix={TileMatrix}&TileRow={TileRow}&TileCol={TileCol}&style=default&format=tiles&tk=3cc6a8542f24dc95c8d1b7dfba10bbd0",
        layer: "img",
        style: "default",
        tileMatrixSetID: "w",
        format: "tiles",
        maximumLevel: 18,
      });
      imageryLayers.addImageryProvider(googleMap);
      this.configScene();
      this.SetInputAction();
      this.get3Dtiles();
    },
    // 配置视窗
    configScene() {
      // Enable lighting based on sun/moon positions(激活基于太阳位置的光照)
      this.$viewer.scene.globe.enableLighting = true;

      // Create an initial camera view
      let initialPosition = new Cesium.Cartesian3.fromDegrees(
        119.62065925,
        32.33722304,
        801.6425
      );
      let initialOrientation = new Cesium.HeadingPitchRoll.fromDegrees(
        41.724684714172732,
        -53.783141852231246,
        0.008819224020548466
      );
      let homeCameraView = {
        destination: initialPosition,
        orientation: {
          heading: initialOrientation.heading,
          pitch: initialOrientation.pitch,
          roll: initialOrientation.roll,
        },
      };
      // Set the initial view
      this.$viewer.scene.camera.setView(homeCameraView);
    },
    SetInputAction() {
      let that = this;
      console.log(that.$viewer);
      var handler = new Cesium.ScreenSpaceEventHandler(
        that.$viewer.scene.canvas
      );
      let viewer = that.$viewer;
      handler.setInputAction(function (movement) {
        var cartesian = viewer.camera.pickEllipsoid(
          movement.endPosition,
          viewer.scene.globe.ellipsoid
        );
        if (cartesian) {
          //                 //将笛卡尔三维坐标转为地图坐标（弧度）
          var cartographic =
            viewer.scene.globe.ellipsoid.cartesianToCartographic(cartesian);
          // //将地图坐标（弧度）转为十进制的度数
          that.Position.lat = Cesium.Math.toDegrees(
            cartographic.latitude
          ).toFixed(4);
          that.Position.log = Cesium.Math.toDegrees(
            cartographic.longitude
          ).toFixed(4);
          that.Position.alt = (
            viewer.camera.positionCartographic.height / 1000
          ).toFixed(2);
        }
      }, Cesium.ScreenSpaceEventType.MOUSE_MOVE);
    },
    get3Dtiles() {
      let viewer = this.$viewer;

      var tileset = viewer.scene.primitives.add(
        new Cesium.Cesium3DTileset({
          url: "http://221.229.121.123:8000/bee/open-63079029343256634/bim/pavementBim/tileset.json",
        })
      );
      // console.log(tileset)
      const transform = this.update3dtilesMaxtrix({
        rx: 0,
        ry: 0,
        rz: -295,
        ps: {
          lat: 32.3340738,
          lng: 119.620305,
          alt: 0,
        },
      });
      // viewer.scene.primitives.add(tileset);

      tileset.readyPromise
        .then(function (tileset) {
          viewer.scene.primitives.add(tileset);
                  tileset._root.transform = transform;
          viewer.zoomTo(
            tileset,
            new Cesium.HeadingPitchRange(
              0.5,
              -0.2,
              tileset.boundingSphere.radius * 1.0
            )
          );
        })
        .otherwise(function (error) {
          console.log(error);
        });
    },
    update3dtilesMaxtrix(options = {}) {
      // const { Cesium } = QE;
      const { rx, ry, rz, ps } = options;
      const mx = Cesium.Matrix3.fromRotationX(Cesium.Math.toRadians(rx));
      const my = Cesium.Matrix3.fromRotationY(Cesium.Math.toRadians(ry));
      const mz = Cesium.Matrix3.fromRotationZ(Cesium.Math.toRadians(rz));
      const rotationX = Cesium.Matrix4.fromRotationTranslation(mx);
      const rotationY = Cesium.Matrix4.fromRotationTranslation(my);
      const rotationZ = Cesium.Matrix4.fromRotationTranslation(mz);
      // 平移 修改经纬度
      const position = Cesium.Cartesian3.fromDegrees(ps.lng, ps.lat, ps.alt);
      const m = Cesium.Transforms.eastNorthUpToFixedFrame(position);
      // 旋转、平移矩阵相乘
      Cesium.Matrix4.multiply(m, rotationX, m);
      Cesium.Matrix4.multiply(m, rotationY, m);
      Cesium.Matrix4.multiply(m, rotationZ, m);
      // 缩放 修改缩放比例
      // var scale = Cesium.Matrix4.fromUniformScale(this.tileModelTool.scale);
      // Cesium.Matrix4.multiply(m, scale, m);
      // 赋值给tileset
      return m;
    },
  },
};
</script>

<style lang="scss" scoped>
#cesiumContainer {
  width: 100vw;
  height: 100vh;
  margin: 0;
  padding: 0;
  overflow: hidden;
}
</style>
