<template>
  <div>
    <p>
      This example demonstrates how to use the <code>onCreated</code> prop to
      obtain access to the VTK render window for one or more component. It also
      shows how to provide an array of vtkVolumes to the component for
      rendering. When we change the RGB Transfer Function for the volume using
      the Window/Level buttons, we can see that this is applied inside both
      components.
    </p>
    <select v-model="selectedFile">
      <option v-for="file in files" :key="file">{{ file }}</option>
    </select>
    <hr />
    <div v-if="loading">
      <h3>Loading...</h3>
    </div>
    <div v-else>
      <h5>Set a Window/Level Preset</h5>
      <div className="btn-group">
        <button className="btn btn-primary" @click="() => setWLPreset('BONE')">
          Bone
        </button>
        <button className="btn btn-primary" @click="() => setWLPreset('HEAD')">
          Head
        </button>
      </div>
      <div class="row">
        <div class="col">
          <view-2d :volumes="volumes" :onCreated="this.saveRenderWindow(0)" />
        </div>
        <div class="col">
          <view-2d :volumes="volumes" :onCreated="this.saveRenderWindow(1)" />
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { View2D, vtkInteractorStyleMPRWindowLevel } from "@/library";
import vtkHttpDataSetReader from "vtk.js/Sources/IO/Core/HttpDataSetReader";
import vtkVolume from "vtk.js/Sources/Rendering/Core/Volume";
import vtkVolumeMapper from "vtk.js/Sources/Rendering/Core/VolumeMapper";

import { files } from "@/components/examples";

const PRESETS = {
  BONE: {
    windowWidth: 100,
    windowCenter: 500
  },
  HEAD: {
    windowWidth: 1000,
    windowCenter: 300
  }
};

export default {
  components: {
    "view-2d": View2D
  },
  data() {
    return {
      volumes: [],
      components: [],
      selectedFile: files[2],
      loading: true
    };
  },
  created() {
    // unreactive internal variables
    this.apis = [];
    this.files = files;
  },
  watch: {
    selectedFile(newVal) {
      this.loadData(newVal);
    }
  },
  methods: {
    setWLPreset(preset) {
      const voi = PRESETS[preset];

      const volume = this.volumes[0];
      const rgbTransferFunction = volume
        .getProperty()
        .getRGBTransferFunction(0);
      const low = voi.windowCenter - voi.windowWidth / 2;
      const high = voi.windowCenter + voi.windowWidth / 2;

      rgbTransferFunction.setMappingRange(low, high);

      this.updateAllViewports();
    },
    updateAllViewports() {
      Object.keys(this.components).forEach(viewportIndex => {
        const component = this.components[viewportIndex];

        component.genericRenderWindow.getRenderWindow().render();
      });
    },
    linkInteractors(renderWindow1, renderWindow2) {
      const i1 = renderWindow1.getInteractor();
      const i2 = renderWindow2.getInteractor();
      const sync = {};

      let src = null;

      function linkOneWay(from, to) {
        from.onStartAnimation(() => {
          if (!src) {
            src = from;
            to.requestAnimation(sync);
          }
        });

        from.onEndAnimation(() => {
          if (src === from) {
            src = null;
            to.cancelAnimation(sync);
            // roughly wait for widgetManager.capture() to finish
            setTimeout(to.render, 1000);
          }
        });
      }

      linkOneWay(i1, i2);
      linkOneWay(i2, i1);
    },
    linkAllInteractors(renderWindows) {
      if (renderWindows.length < 2) {
        return;
      }

      for (let i = 0; i < renderWindows.length - 1; i++) {
        for (let j = i + 1; j < renderWindows.length; j++) {
          this.linkInteractors(renderWindows[i], renderWindows[j]);
        }
      }
    },
    saveRenderWindow(viewportIndex) {
      return component => {
        this.$set(this.components, viewportIndex, component);

        if (viewportIndex === 1) {
          const renderWindow = component.genericRenderWindow.getRenderWindow();

          // TODO: This is a hacky workaround because disabling the vtkInteractorStyleMPRSlice is currently
          // broken. The camera.onModified is never removed.
          renderWindow
            .getInteractor()
            .getInteractorStyle()
            .setVolumeMapper(null);

          const istyle = vtkInteractorStyleMPRWindowLevel.newInstance();

          renderWindow.getInteractor().setInteractorStyle(istyle);
          // console.log(
          //   "setting component",
          //   viewportIndex,
          //   component,
          //   component.volumes
          // );
          istyle.setVolumeMapper(component.volumes[0]);

          const renderWindows = Object.values(this.components).map(a =>
            a.genericRenderWindow.getRenderWindow()
          );
          this.linkAllInteractors(renderWindows);
        }
      };
    },
    loadData(fileString) {
      this.loading = true;
      const reader = vtkHttpDataSetReader.newInstance({
        fetchGzip: true
      });
      const volumeActor = vtkVolume.newInstance();
      const volumeMapper = vtkVolumeMapper.newInstance();

      volumeActor.setMapper(volumeMapper);

      reader
        .setUrl(`/${fileString || this.selectedFile}`, { loadData: true })
        .then(() => {
          const data = reader.getOutputData();
          volumeMapper.setInputData(data);
          this.volumes = [volumeActor];
          this.loading = false;
        });
    }
  },
  mounted() {
    this.loadData();
  }
};
</script>

<style></style>
