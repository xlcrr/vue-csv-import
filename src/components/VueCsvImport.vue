<template>
  <div class="vue-csv-uploader">
    <div class="form">
      <div class="vue-csv-uploader-part-one">
        <div class="form-check form-group csv-import-checkbox" v-if="headers === null" style="margin-bottom: 1em;">
          <slot name="hasHeaders" :headers="hasHeaders" :toggle="toggleHasHeaders">
            <input :class="checkboxClass" type="checkbox" :id="makeId('hasHeaders')" :value="hasHeaders" @change="toggleHasHeaders">
            <label class="form-check-label" :for="makeId('hasHeaders')">
                File Has Headers
            </label>
          </slot>
        </div>
        <div class="form-group csv-import-file" style="margin-bottom: 1em;">
          <div class="file has-name">
            <label class="file-label">
              <input ref="csv" type="file" @change.prevent="validFileMimeType" class="file-input" name="csv">
              <span class="file-cta">
                <span class="file-icon">
                  <i class="fas fa-upload"></i>
                </span>
                <span class="file-label">
                  Choose a file…
                </span>
              </span>
              <span class="file-name" :key="filename">
                {{ this.filename }}
              </span>
            </label>
          </div>

          <slot name="error" v-if="showErrorMessage">
            <div class="invalid-feedback d-block">
                File type is invalid
            </div>
          </slot>
        </div>
        <div class="form-group">
          <slot name="next" :load="load">
            <button :disabled="disabledNextButton" class="button is-medium is-primary" @click.prevent="load">
              {{ loadBtnText }}
            </button>
          </slot>
        </div>
      </div>
      <div class="vue-csv-uploader-part-two">
        <div class="vue-csv-mapping" v-if="sample">
          <table :class="tableClass">
            <slot name="thead">
              <thead>
              <tr>
                <th>Field</th>
                <th>CSV Column</th>
              </tr>
              </thead>
            </slot>
            <tbody>
              <tr v-for="(field, key) in fieldsToMap" :key="key">
                <td>{{ field.label }}</td>
                <td>
                  <select class="input" :name="`csv_uploader_map_${key}`" v-model="map[field.key]">
                    <option value="" disabled hidden>Select a column</option>
                    <option v-for="(column, key) in firstRow" :key="key" :value="key">{{ column }}</option>
                  </select>
                </td>
              </tr>
            </tbody>
          </table>
          <div class="form-group" v-if="url">
            <slot name="submit" :submit="submit">
              <button class="button is-medium is-primary" @click.prevent="submit" :value="submitBtnText">
                Upload
              </button>
            </slot>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { drop, every, forEach, get, isArray, map, set } from 'lodash';
import axios from 'axios';
import Papa from 'papaparse';
import mimeTypes from "mime-types";

export default {
    props: {
        value: Array,
        url: {
            type: String
        },
        mapFields: {
            required: true
        },
        callback: {
            type: Function,
            default: () => ({})
        },
        catch: {
            type: Function,
            default: () => ({})
        },
        finally: {
            type: Function,
            default: () => ({})
        },
        parseConfig: {
            type: Object,
            default() {
                return {};
            }
        },
        headers: {
            default: null
        },
        loadBtnText: {
            type: String,
            default: "Next"
        },
        submitBtnText: {
            type: String,
            default: "Submit"
        },
        tableClass: {
            type: String,
            default: "table"
        },
        checkboxClass: {
            type: String,
            default: "form-check-input"
        },
        buttonClass: {
            type: String,
            default: "btn btn-primary"
        },
        inputClass: {
            type: String,
            default: "file-input"
        },
        validation: {
            type: Boolean,
            default: true,
        },
        fileMimeTypes: {
            type: Array,
            default: () => {
                return ["text/csv", "text/x-csv", "application/vnd.ms-excel", "text/plain"];
            }
        }
    },

    data: () => ({
        form: {
            csv: null,
        },
        fieldsToMap: [],
        map: {},
        hasHeaders: true,
        csv: null,
        sample: null,
        isValidFileMimeType: false,
        fileSelected: false,
        filename: ''
    }),

    created() {
        this.hasHeaders = this.headers;

        if (isArray(this.mapFields)) {
            this.fieldsToMap = map(this.mapFields, (item) => {
                return {
                    key: item,
                    label: item
                };
            });
        } else {
            this.fieldsToMap = map(this.mapFields, (label, key) => {
                return {
                    key: key,
                    label: label
                };
            });
        }
    },
    
    buildMappedCsv() {

        const _this = this;

        let csv = this.hasHeaders ? drop(this.csv) : this.csv;

        return map(csv, (row) => {
            let newRow = {};

            forEach(_this.map, (column, field) => {
                set(newRow, field, get(row, column));
            });

            return newRow;
        });
    },
    
    validFileMimeType() {
        let file = this.$refs.csv.files[0];
        const mimeType = file.type === "" ? mimeTypes.lookup(file.name) : file.type;

            this.readFile((output) => {
                _this.sample = get(Papa.parse(output, { preview: 2, skipEmptyLines: true }), "data");
                _this.csv = get(Papa.parse(output, { skipEmptyLines: true }), "data");
            });
        },

        readFile(callback) {
          let file = this.$refs.csv.files[0];

          this.readFile((output) => {
              _this.sample = _.get(Papa.parse(output, { preview: 2, skipEmptyLines: true }), "data");
              _this.csv = _.get(Papa.parse(output, { skipEmptyLines: true }), "data");
          });
        },
        
        readFile(callback) {
            let file = this.$refs.csv.files[0];

            if (file) {
                let reader = new FileReader();
                reader.readAsText(file, "UTF-8");
                reader.onload = function (evt) {
                    callback(evt.target.result);
                };
                reader.onerror = function () {
                };
            }
        },
        watch: {
            map: {
                deep: true,
                handler: function (newVal) {
                    if (!this.url) {
                        let hasAllKeys = Array.isArray(this.mapFields) ? every(this.mapFields, function (item) {
                            return newVal.hasOwnProperty(item);
                        }) : every(this.mapFields, function (item, key) {
                            return newVal.hasOwnProperty(key);
                        });
                    if (hasAllKeys) {
                        this.submit();
                    }
                }
            }
        }
    },
    computed: {
        firstRow() {
            return _.get(this, "sample.0");
        },
        computed: {
            firstRow() {
                return get(this, "sample.0");
            },
            showErrorMessage() {
                return this.fileSelected && !this.isValidFileMimeType;
            },
            disabledNextButton() {
                return !this.isValidFileMimeType;
            }
        },
        disabledNextButton() {
            return !this.isValidFileMimeType;
        }
    },
};
</script>
