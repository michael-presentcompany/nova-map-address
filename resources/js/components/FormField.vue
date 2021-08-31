<template>
<default-field :field="field">
<template slot="field">
  <div class="google-map w-full" :id="mapName" />

  <br>

  <div class="w-full">
    <vue-google-autocomplete
        :id="map"
        :classname="form-control"
        :placeholder="field.name"
    >
      <input
          :id="field.name"
          type="text"
          class="w-full form-control form-input form-input-bordered"
          :class="errorClasses"
          :placeholder="field.name"
          :value="field.name"
          v-model="value"
      />
    </vue-google-autocomplete>
  </div>

  <br>

  <div class="flex flex-wrap w-full">
    <div class="flex w-1/2">
      <div class="w-1/5 py-3 pl-2">
        <label class="inline-block text-80 pt-2 leading-tight" :for="field.latitude">Lat</label>
      </div>
      <div class="py-3 w-4/5">
        <input :id="field.latitude" type="text"
               class="w-full form-control form-input form-input-bordered"
               :class="errorClasses"
               :placeholder="field.latitude"
               v-model="field.lat"
               value="field.latitude"
        />
      </div>
    </div>
    <div class="flex w-1/2">
      <div class="w-1/5 py-3 pl-2">
        <label class="inline-block text-80 pt-2 leading-tight" :for="field.longitude">Lng</label>
      </div>
      <div class="py-3 w-4/5">
        <input :id="field.longitude" type="text"
               class="w-full form-control form-input form-input-bordered"
               :class="errorClasses"
               :placeholder="field.longitude"
               v-model="field.lng"
               value="field.longitude"
        />
      </div>
    </div>
  </div>
  <p v-if="hasError" class="my-2 text-danger">
    {{ firstError }}
  </p>
</template>
</default-field>
</template>

<style scoped>
.google-map {
  width: 100%;
  height: 300px;
  margin: 0 auto;
  background: gray;
  border:solid 1px #ccc;
}
</style>

<script>
import { FormField, HandlesValidationErrors } from 'laravel-nova';
import VueGoogleAutocomplete from 'vue-google-autocomplete';

export default {
  name: 'google-map',
  mixins: [FormField, HandlesValidationErrors],
  props: ['resourceName', 'resourceId', 'field'],
  data() {
    return {
      mapName: `${this.name || 'gem'}-map`,
      latLngResult: undefined
    }
  },

  mounted() {
    let address = this;

    let initialValues = !!this.field && !!this.field.value ?
        JSON.parse(this.field.value) :
        { formatted_address: 'Sydney NSW, Australia', lat: -33.8688197, lng: 151.2092955 };

    let lat = this.field.lat || this.field.initial_lat || initialValues.lat;
    let lng = this.field.lng || this.field.initial_lng || initialValues.lng;

    this.field.latitude = lat;
    this.field.longitude = lng;
    this.value = initialValues.formatted_address;

    const element = document.getElementById(this.mapName);
    const mapsLatLng = new google.maps.LatLng(lat, lng);

    const map = new google.maps.Map(element, {
      zoom: this.field.zoom || 4,
      center: mapsLatLng
    });

    const marker = new google.maps.Marker({
      position: mapsLatLng,
      map
    });

    const geocoder = new google.maps.Geocoder();

    const places = new google.maps.places.Autocomplete(
        document.getElementById(this.field.name),
        { types: ["geocode"] }
    );

    places.addListener('place_changed', () => {
      const place = places.getPlace();
      const placeLat = place.geometry.location.lat();
      const placeLng = place.geometry.location.lng();

      this.fillInLatLng({
        address: place.formatted_address,
        lat: placeLat,
        lng: placeLng,
        marker,
        map
      })
    });

    google.maps.event.addListener(map, 'click', (event) => {
      marker.setPosition(event.latLng);

      geocoder.geocode({'location': event.latLng}, (results, status) => {
        if (status === google.maps.GeocoderStatus.OK) {
          let result = results[0];

          if (result) {
            address.value = result.formatted_address;
            address.field.lat = event.latLng.lat().toFixed(6);
            address.field.lng = event.latLng.lng().toFixed(6);

            this.fillInLatLng({
              address: result.formatted_address,
              lat: event.latLng.lat(),
              lng: event.latLng.lng()
            });
          } else {
            console.error('No results found');
          }
        } else {
          console.error(`Geocoder failed due to: ${status}`);
        }
      });
    });
  },

  methods: {
    setInitialValue() {
      this.value = this.field.value || ''
    },

    fill(formData) {
      formData.append(this.field.attribute, this.latLngResult || '');
      formData.append(this.field.latitude, this.field.lat || '');
      formData.append(this.field.longitude, this.field.lng || '');
    },

    handleChange(value) {
      this.value = value;
    },

    fillInLatLng({ address, lat, lng, marker, map }) {
      this.field.latitude = lat;
      this.field.longitude = lng;
      this.latLngResult = JSON.stringify({formatted_address: address, lat, lng});
      this.value = address;

      !!marker && marker.setPosition(new google.maps.LatLng( lat, lng ));
      !!map && map.panTo(new google.maps.LatLng( lat, lng ));
    },
  },
}
</script>