<template>
  <div id="patient-module" class="mx-2 height-100">
    <div id="patient-filter-controls">
      <v-select
        v-model="patientName"
        :items="patientListWithImageEntry"
        item-text="name"
        item-value="id"
        dense
        filled
        single-line
        hide-details
        label="Patient"
        prepend-icon="mdi-account"
        no-data-text="No patients loaded"
        placeholder="Select a patient"
        class="no-select"
      />
    </div>
    <div id="patient-data-list">
      <item-group :value="selectedBaseImage" @change="setSelection">
        <template v-if="!patientName"> No patient selected </template>
        <template v-else-if="patientName === IMAGES">
          <div v-if="imageList.length === 0">No non-dicom images available</div>
          <groupable-item
            v-for="imgID in imageList"
            :key="imgID"
            v-slot="{ active, select }"
            :value="imgID"
          >
            <avatar-list-card
              :active="active"
              :title="dataIndex[imgID].name"
              :image-url="getImageThumbnail(imgID)"
              :image-size="100"
              @click="select"
            >
              <div class="body-2 text-truncate">
                {{ dataIndex[imgID].name }}
              </div>
              <div class="caption">
                Dims: ({{ dataIndex[imgID].dims.join(', ') }})
              </div>
              <div class="caption">
                Spacing: ({{
                  dataIndex[imgID].spacing.map((s) => s.toFixed(2)).join(', ')
                }})
              </div>
              <template v-slot:menu>
                <v-list>
                  <v-list-item @click.stop="removeData(imgID)">
                    Delete
                  </v-list-item>
                </v-list>
              </template>
            </avatar-list-card>
          </groupable-item>
        </template>
        <template v-else>
          <v-expansion-panels id="patient-data-studies" accordion multiple>
            <v-expansion-panel
              v-for="study in visibleStudies"
              :key="study.StudyInstanceUID"
              class="patient-data-study-panel"
            >
              <v-expansion-panel-header
                color="#1976fa0a"
                class="no-select"
                :title="study.StudyDate"
              >
                <v-icon class="ml-n3 pr-3">mdi-folder-table</v-icon>
                <div class="study-header">
                  <div class="subtitle-2 study-header-line">
                    {{
                      study.StudyDescription ||
                      study.StudyDate ||
                      study.StudyInstanceUID
                    }}
                  </div>
                  <div
                    v-if="study.StudyDescription"
                    class="caption study-header-line"
                  >
                    {{ study.StudyDate }}
                  </div>
                </div>
              </v-expansion-panel-header>
              <v-expansion-panel-content>
                <div class="my-2 volume-list">
                  <groupable-item
                    v-for="volInfo in getVolumesForStudy(
                      study.StudyInstanceUID
                    )"
                    :key="volInfo.VolumeID"
                    v-slot:default="{ active, select }"
                    :value="dicomVolumeToDataID[volInfo.VolumeID]"
                  >
                    <v-card
                      outlined
                      ripple
                      :color="active ? 'light-blue lighten-4' : ''"
                      class="volume-card"
                      :title="volInfo.SeriesDescription"
                      @click="select"
                    >
                      <v-img
                        contain
                        height="100px"
                        :src="dicomThumbnails[volInfo.VolumeID]"
                      />
                      <v-card-text
                        class="text--primary caption text-center series-desc mt-n3"
                      >
                        <div>[{{ volInfo.NumberOfSlices }}]</div>
                        <div class="text-ellipsis">
                          {{ volInfo.SeriesDescription || '(no description)' }}
                        </div>
                        <div class="actions">
                          <v-btn
                            small
                            icon
                            @click.stop="
                              removeData(dicomVolumeToDataID[volInfo.VolumeID])
                            "
                          >
                            <v-icon>mdi-delete</v-icon>
                          </v-btn>
                        </div>
                      </v-card-text>
                    </v-card>
                  </groupable-item>
                </div>
              </v-expansion-panel-content>
            </v-expansion-panel>
          </v-expansion-panels>
        </template>
      </item-group>
    </div>
  </div>
</template>

<script>
import { mapState, mapActions } from 'vuex';

import ItemGroup from '@/src/components/ItemGroup.vue';
import GroupableItem from '@/src/components/GroupableItem.vue';
import AvatarListCard from '@/src/components/AvatarListCard.vue';

import { DataTypes, NO_SELECTION } from '@/src/constants';

const IMAGES = Symbol('IMAGES');
const canvas = document.createElement('canvas');

// Assume itkImage type is Uint8Array
function itkImageToURI(itkImage) {
  const [width, height] = itkImage.size;
  const im = new ImageData(width, height);
  const arr32 = new Uint32Array(im.data.buffer);
  const itkBuf = itkImage.data;
  for (let i = 0; i < itkBuf.length; i += 1) {
    const byte = itkBuf[i];
    // ABGR order
    // eslint-disable-next-line no-bitwise
    arr32[i] = (255 << 24) | (byte << 16) | (byte << 8) | byte;
  }

  canvas.width = width;
  canvas.height = height;
  const ctx = canvas.getContext('2d');
  ctx.putImageData(im, 0, 0);
  return canvas.toDataURL('image/png');
}

function scalarImageToURI(values, width, height, scaleMin, scaleMax) {
  const im = new ImageData(width, height);
  const arr32 = new Uint32Array(im.data.buffer);
  // scale to 1 unsigned byte
  const factor = 255 / (scaleMax - scaleMin);
  for (let i = 0; i < values.length; i += 1) {
    const byte = Math.floor((values[i] - scaleMin) * factor);
    // ABGR order
    // eslint-disable-next-line no-bitwise
    arr32[i] = (255 << 24) | (byte << 16) | (byte << 8) | byte;
  }

  canvas.width = width;
  canvas.height = height;
  const ctx = canvas.getContext('2d');
  ctx.putImageData(im, 0, 0);
  return canvas.toDataURL('image/png');
}

export default {
  name: 'PatientBrowser',

  components: {
    ItemGroup,
    GroupableItem,
    AvatarListCard,
  },

  data() {
    return {
      patientName: '',
      imageThumbnails: {}, // dataID -> Image
      dicomThumbnails: {}, // volumeID -> Image
      pendingDicomThumbnails: {},

      IMAGES, // symbol
    };
  },

  computed: {
    ...mapState({
      selectedBaseImage: 'selectedBaseImage',
      dicomVolumeToDataID: 'dicomVolumeToDataID',
      imageList: (state) => state.data.imageIDs,
      dataIndex: (state) => state.data.index,
      vtkCache: (state) => state.data.vtkCache,
    }),
    ...mapState('dicom', {
      patientIndex: 'patientIndex',
      patientStudies: 'patientStudies',
      studyIndex: 'studyIndex',
      studyVolumes: 'studyVolumes',
      volumeIndex: 'volumeIndex',
    }),
    patients(state) {
      const seen = new Set();
      const patients = [];

      // select key, name from patientIndex group by name
      Object.entries(state.patientIndex).forEach(([key, p]) => {
        const name = p.PatientName;
        if (!seen.has(name)) {
          seen.add(name);
          patients.push([key, p]);
        }
      });

      patients.sort((a, b) => a[1].PatientName < b[1].PatientName);
      return patients.map(([key, p]) => ({
        id: p.PatientName,
        name: p.PatientName,
        key, // patient key
        patient: p,
      }));
    },
    patientListWithImageEntry() {
      return [].concat(
        {
          id: IMAGES,
          name: 'Non-DICOM images',
          key: null,
          patient: null,
        },
        this.patients
      );
    },
    visibleStudies() {
      // select all studies associated with a patient whose name is
      // the same as this.patientName
      let studies = [];
      this.patients
        .filter(({ patient }) => patient.PatientName === this.patientName)
        .forEach(({ key }) => {
          studies = studies.concat(
            this.patientStudies[key].map(
              (studyKey) => this.studyIndex[studyKey]
            )
          );
        });
      return studies;
    },
  },

  watch: {
    selectedBaseImage(id) {
      if (id !== NO_SELECTION) {
        const dataInfo = this.dataIndex[id];
        if (dataInfo.type === DataTypes.Image) {
          this.patientName = IMAGES;
        } else if (dataInfo.type === DataTypes.Dicom) {
          const patientInfo = this.patientIndex[dataInfo.patientKey];
          this.patientName = patientInfo.PatientName;
        }
      }
    },
    patients() {
      // if patient index is updated, then try to select first one
      if (!this.patientName) {
        if (this.patients.length) {
          this.patientName = this.patients[0].id;
        } else {
          this.patientName = '';
        }
      }
    },
  },

  methods: {
    getVolumesForStudy(studyUID) {
      const volumeList = (this.studyVolumes[studyUID] ?? []).map(
        (volID) => this.volumeIndex[volID]
      );

      // trigger a background job fetch thumbnails
      this.doBackgroundDicomThumbnails(volumeList);

      return volumeList;
    },

    async setSelection(sel) {
      if (sel !== this.selectedBaseImage) {
        await this.selectBaseImage(sel);
        await this.updateScene({ reset: true });
      }
    },

    async doBackgroundDicomThumbnails(volumeList) {
      volumeList.forEach(async (volInfo) => {
        const id = volInfo.VolumeID;
        if (
          !(id in this.dicomThumbnails || id in this.pendingDicomThumbnails)
        ) {
          this.$set(this.pendingDicomThumbnails, id, true);
          try {
            const middleSlice = Math.round(Number(volInfo.NumberOfSlices) / 2);
            const thumbItkImage = await this.getVolumeSlice({
              volumeID: id,
              slice: middleSlice,
              asThumbnail: true,
            });
            this.$set(this.dicomThumbnails, id, itkImageToURI(thumbItkImage));
          } finally {
            delete this.pendingDicomThumbnails[id];
          }
        }
      });
    },

    getImageThumbnail(id) {
      if (id in this.imageThumbnails) {
        return this.imageThumbnails[id];
      }
      if (id in this.vtkCache) {
        const imageData = this.vtkCache[id];
        const scalars = imageData.getPointData().getScalars();
        const dims = imageData.getDimensions();
        const length = dims[0] * dims[1];
        const sliceOffset = Math.floor(dims[2] / 2) * length;
        const slice = scalars
          .getData()
          .subarray(sliceOffset, sliceOffset + length);
        const dataRange = scalars.getRange();

        const img = scalarImageToURI(
          slice,
          dims[0],
          dims[1],
          dataRange[0],
          dataRange[1]
        );
        this.$set(this.imageThumbnails, id, img);
        return img;
      }
      return '';
    },

    ...mapActions(['selectBaseImage', 'removeData']),
    ...mapActions('visualization', ['updateScene']),
    ...mapActions('dicom', ['getVolumeSlice']),
  },
};
</script>

<style>
#patient-data-studies .v-expansion-panel--active > .v-expansion-panel-header {
  min-height: unset;
}

#patient-data-studies .v-expansion-panel-content__wrap {
  padding: 0 8px;
}

#patient-data-studies .v-expansion-panel::before {
  box-shadow: none;
}
</style>

<style scoped>
#patient-module {
  display: flex;
  flex-flow: column;
}

#patient-filter-controls {
  border-bottom: 1px solid rgba(0, 0, 0, 0.12);
  padding-bottom: 12px;
}

#patient-data-list {
  flex: 2;
  margin-top: 12px;
  overflow-y: scroll;
}

.patient-data-study-panel {
  border: 1px solid #ddd;
}

.volume-list {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(140px, 1fr));
  grid-template-rows: 180px;
  justify-content: center;
}

.volume-card {
  padding: 8px;
  cursor: pointer;
}

.study-header {
  width: 100%;
  overflow: hidden;
  white-space: nowrap;
}

.study-header-line {
  width: 100%;
  text-overflow: ellipsis;
  overflow: hidden;
}

.series-desc {
  white-space: nowrap;
  text-overflow: ellipsis;
  overflow: hidden;
}
</style>
