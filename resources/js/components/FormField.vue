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
      placeResult: undefined
    }
  },
  mounted() {
    let address = this;
    let lat = this.field.lat || this.field.initial_lat || 38.261842;
    let lng = this.field.lng ||this.field.initial_lng || -0.6868031;

    this.field.latitude = lat;
    this.field.longitude = lng;

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

    map.setCenter(mapsLatLng);
    marker.setPosition(mapsLatLng);

    places.addListener('place_changed', () => {
      const place = places.getPlace();
      const placeLat = place.geometry.location.lat();
      const placeLng = place.geometry.location.lng();

      this.fillInLatLng({
        place: {
          address_components: place.address_components,
          formatted_address: place.formatted_address,
          adr_address: place.adr_address,
          place_id: place.place_id,
          url: place.url,
          lat: placeLat,
          lng: placeLng
        },
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
      formData.append(this.field.attribute, this.placeResult || '');
      formData.append(this.field.latitude, this.field.lat || '');
      formData.append(this.field.longitude, this.field.lng || '');
    },

    handleChange(value) {
      this.value = value;
    },

    fillInLatLng({ address, place, lat, lng, marker, map }) {
      this.field.latitude = lat;
      this.field.longitude = lng;
      this.placeResult = JSON.stringify(place);
      this.value = address;

      marker.setPosition(new google.maps.LatLng( lat, lng ));
      map.panTo(new google.maps.LatLng( lat, lng ));
    },
  },
}
</script>