<template>
  <div id="app">
    <div class="navbar">
      <div class="app-heading">
        <h1>Image Gallery</h1>
      </div>

      <div class="app-search">
        <input
          v-model="searchQuery"
          :class="{ 'app-search-input__has-history': historyNotEmpty }"
          type="text"
          placeholder="Search"
          @focus="searchHistoryHidden = false"
          @blur="searchHistoryHidden = true"
        />

        <a class="app-search-input-pin" @click="addPin(searchQuery)">
          <i class="flaticon-push-pin" />
        </a>

        <div
          v-if="historyNotEmpty"
          class="app-search-history"
          :style="{ display: searchHistoryHidden ? 'none' : 'block' }"
        >
          <nav class="app-search-history-nav">
            <a
              v-for="(item, key) in searchHistory"
              :key="key"
              class="app-search-history-item"
              @mousedown="searchQuery = item"
            >
              {{ item }}
            </a>
          </nav>
        </div>
      </div>
      <div v-if="searchPinnedNotEmpty" class="app-search-pins">
        <nav class="app-search-pins-nav">
          <a
            v-for="(pin, key) in searchPinned"
            :key="key"
            class="app-search-pins-item"
          >
            {{ pin }}
            <a class="app-search-pins-item__close" @click="deletePin(key)">
              X
            </a>
          </a>
        </nav>
      </div>
    </div>
    <div class="image-wrapper">
      <div v-if="isLoading" class="image-wrapper__loading">
        Loading...
      </div>

      <div v-if="error" class="image-wrapper__error">
        {{ error }}
      </div>

      <div v-if="images" class="image-wrapper__loaded">
        <a
          v-for="(image, index) in images"
          :key="index"
          :href="image.url_o ? image.url_o : image.url_z"
          class="image-item"
        >
          <img
            :src="image.url_z ? image.url_z : image.url_o"
            :alt="image.title"
          />
        </a>
      </div>
      <infinite-loading @infinite="infiniteLoad" />
    </div>
  </div>
</template>

<script>
import _ from "lodash";
import InfiniteLoading from "vue-infinite-loading";

export default {
  data() {
    return {
      isLoading: true,
      error: undefined,
      images: [],
      searchQuery: "",
      searchHistory: [],
      searchHistoryHidden: true,
      searchPinned: [],
      page: 2
    };
  },
  components: {
    InfiniteLoading
  },
  methods: {
    loadData() {
      let url =
        "https://api.flickr.com/services/rest/?method=flickr.photos.getRecent&format=json&nojsoncallback=1&api_key=bac9f1ccfd854f27894fd47c4f01b1e8&per_page=20&extras=url_o,url_z";

      this.error = undefined;
      this.images = [];
      this.isLoading = true;

      fetch(url, {
        method: "GET",
        cache: "default"
      })
        .then(res => {
          if (res.ok) {
            this.isLoading = false;
            return res.json();
          }
          this.error = "Something was wrong :(";
        })
        .then(data => {
          this.images = data.photos.photo;
        })
        .catch(err => {
          this.isLoading = false;
          this.error = err.toString();
        });
    },
    searchData() {
      this.isLoading = true;
      this.error = undefined;
      this.images = [];

      let query = "";
      if (this.searchPinnedNotEmpty) {
        query = this.searchPinned.join(",");
      } else {
        query = this.searchQuery;
      }

      let searchUrl = `https://api.flickr.com/services/rest/?method=flickr.photos.search&content_type=1&safe_search=1&per_page=20&format=json&nojsoncallback=1&api_key=bac9f1ccfd854f27894fd47c4f01b1e8&content_type=1&is_getty=1&tags=${query}&extras=url_o,url_z&media=photos`;

      if (
        this.searchQuery != "" &&
        !this.searchHistory.includes(this.searchQuery)
      ) {
        this.searchHistory.push(this.searchQuery);
      }

      if (!_.isEmpty(this.searchHistory)) {
        localStorage.setItem(
          "searchHistory",
          JSON.stringify(this.searchHistory)
        );
      }

      fetch(searchUrl, {
        method: "GET",
        cache: "default"
      })
        .then(res => {
          if (res.ok) {
            this.isLoading = false;
            return res.json();
          }
          this.error = "Something was wrong :(";
        })
        .then(data => {
          if (!_.isEmpty(data.photos.photo)) {
            this.images = data.photos.photo;
          } else {
            this.error = "No results found :(";
          }
        })
        .catch(err => {
          this.isLoading = false;
          this.error = err.toString();
        });
    },
    infiniteLoad($state) {
      let url = "";
      let query = "";

      if (this.searchPinnedNotEmpty) {
        query = this.searchPinned.join(",");
      } else {
        query = this.searchQuery;
      }

      if (query === "") {
        url = `https://api.flickr.com/services/rest/?method=flickr.photos.getRecent&format=json&nojsoncallback=1&api_key=bac9f1ccfd854f27894fd47c4f01b1e8&per_page=20&extras=url_o,url_z&page=${
          this.page
        }`;
      } else {
        url = `https://api.flickr.com/services/rest/?method=flickr.photos.search&content_type=1&safe_search=1&per_page=20&format=json&nojsoncallback=1&api_key=bac9f1ccfd854f27894fd47c4f01b1e8&content_type=1&is_getty=1&tags=${query}&extras=url_o,url_z&media=photos&page=${
          this.page
        }`;
      }

      fetch(url, {
        method: "GET"
      })
        .then(data => data.json())
        .then(data => {
          if (data.photos.photo.length) {
            this.page += 1;
            this.images.push(...data.photos.photo);
            $state.loaded();
          } else {
            $state.complete();
          }
        });
    },
    addPin(item) {
      if (
        !_.isEmpty(item) &&
        item != "" &&
        !this.searchPinned.includes(item) &&
        this.searchPinned.length < 2
      ) {
        this.searchPinned.push(item);
        this.searchData();
      }
    },
    deletePin(itemKey) {
      this.searchPinned.splice(itemKey, 1);
      this.searchData();
    }
  },
  computed: {
    historyNotEmpty: function() {
      if (!_.isEmpty(this.searchHistory)) {
        return true;
      } else {
        return false;
      }
    },
    searchPinnedNotEmpty: function() {
      if (!_.isEmpty(this.searchPinned)) {
        return true;
      } else {
        return false;
      }
    }
  },
  watch: {
    searchQuery: function() {
      this.error = undefined;
      this.images = [];
      this.isLoading = true;
      this.page = 2;
      this.debounceSearchData();
    }
  },
  created() {
    this.loadData();
    this.debounceSearchData = _.debounce(this.searchData, 1000);

    if (localStorage.getItem("searchHistory")) {
      this.searchHistory = JSON.parse(localStorage.getItem("searchHistory"));
    }
  }
};
</script>

<style lang="scss">
@import url("https://fonts.googleapis.com/css?family=Rubik:300,400,500");
@import "./assets/_reset.scss";
body {
  font-family: "Rubik", sans-serif;
  font-size: 16px;
  background: #fafafa;
}
.navbar {
  display: flex;
  flex-direction: column;
  align-items: center;
  background: #fff;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.5);
}
.app-heading {
  padding: 16px 0;
  h1 {
    margin: 0;
    text-align: center;
  }
}
.app-search {
  position: relative;
  max-width: 300px;
  display: flex;
  justify-content: center;
  padding-top: 8px;
  padding-bottom: 12px;
  input {
    height: 48px;
    padding: 12px 32px;
    border: 0;
    outline: none;
    border-radius: 24px;
    background: #fff;
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.35);
    transition: box-shadow 0.3s ease-in-out;
    box-sizing: border-box;
  }
  input:focus {
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.5);
  }
  .app-search-input-pin {
    position: absolute;
    right: 6px;
    top: 12px;
    color: #424242;
    border-radius: 50%;
    padding: 8px;
    cursor: pointer;
    transition: all linear 0.1s;
  }
  .app-search-input-pin:hover {
    background: rgba(0, 0, 0, 0.1);
    transform: rotateZ(45deg);
  }
  .app-search-input__has-history:focus {
    border-bottom-left-radius: 0;
    border-bottom-right-radius: 0;
  }
  .app-search-history {
    display: none;
    position: absolute;
    left: 0;
    top: 54px;
    width: 100%;
    z-index: 10;
    background: #fff;
    border-bottom-left-radius: 24px;
    border-bottom-right-radius: 24px;
    box-shadow: 0 6px 8px rgba(0, 0, 0, 0.35);
    .app-search-history-nav {
      display: flex;
      flex-direction: column;
      margin-top: 20px;
      margin-bottom: 24px;
      max-height: 360px;
      overflow-x: hidden;
      overflow-y: scroll;
      .app-search-history-item {
        cursor: pointer;
        padding: 12px 16px;
        text-decoration: none;
        color: #424242;
      }
      .app-search-history-item:hover {
        background: rgba(0, 0, 0, 0.1);
      }
    }
    .app-search-history-nav::before {
      position: absolute;
      top: 4px;
      left: 12px;
      content: "Last search queries";
      font-size: 12px;
      color: #9e9e9e;
    }
  }
}
.app-search-pins {
  margin: 16px 0;
  .app-search-pins-nav {
    .app-search-pins-item {
      margin-right: 16px;
      text-transform: uppercase;
      background: #43a047;
      padding: 4px 8px;
      border-radius: 22px;
      font-weight: 500;
      color: #fff;
      box-shadow: 0 6px 8px rgba(0, 0, 0, 0.35);
    }
    .app-search-pins-item:last-child {
      margin-right: 0;
    }
    .app-search-pins-item__close {
      margin-left: 8px;
      font-size: 14px;
      cursor: pointer;
      color: rgba(255, 255, 255, 0.6);
    }
    .app-search-pins-item__close:hover {
      color: rgba(255, 255, 255, 0.95);
    }
  }
}
.image-wrapper {
  max-width: 1200px;
  margin: 0 auto;
  padding: 12px 0;
  .image-wrapper__loaded {
    display: flex;
    flex-wrap: wrap;
    .image-item {
      position: relative;
      display: flex;
      justify-content: center;
      align-items: center;
      flex: 1 0 24%;
      max-height: 250px;
      min-width: 250px;
      margin: 4px;
      overflow: hidden;
    }
    .image-item::after {
      content: "";
      width: 100%;
      height: 100%;
      position: absolute;
      top: 0;
      left: 0;
      background: rgba(0, 0, 0, 0);
      transition: background linear 0.3s;
    }
    .image-item:hover::after {
      content: "";
      width: 100%;
      height: 100%;
      position: absolute;
      top: 0;
      left: 0;
      background: rgba(0, 0, 0, 0.35);
    }
  }
}
@media screen and (max-width: 768px) {
  .image-item {
    flex: 1 0 49%;
  }
}
</style>
