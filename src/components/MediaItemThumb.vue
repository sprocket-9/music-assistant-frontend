<!-- TODO: Restore fallback image based on media type -->
<template>
  <img
    ref="imageTag"
    :loading="$props.lazy"
    :height="$props.size || $props.height || '600px'"
    :width="$props.size || $props.width || '600px'"
    :src="imgData"
    :class="{ rounded: rounded }"
    :style="lazyStyle"
  />
</template>

<script setup lang="ts">
import { computed, ref, onMounted } from 'vue';
import type {
  ItemMapping,
  MediaItemImage,
  MediaItemType,
  QueueItem,
} from '@/plugins/api/interfaces';
import { ImageType, MediaType } from '@/plugins/api/interfaces';
import { api } from '@/plugins/api';
import { useTheme } from 'vuetify';
import {
  imgCoverDark,
  imgCoverLight,
  iconFolder,
} from '@/components/QualityDetailsBtn.vue';

export interface Props {
  item?: MediaItemType | ItemMapping | QueueItem;
  width?: string | number;
  height?: string | number;
  size?: string | number;
  aspectRatio?: string | number;
  cover?: boolean;
  fallback?: string;
  thumb?: boolean;
  lazy?: HTMLImageElement['loading'];
  rounded?: boolean;
}

const props = withDefaults(defineProps<Props>(), {
  item: undefined,
  size: undefined,
  width: '100%',
  height: 'auto',
  aspectRatio: '1/1',
  cover: true,
  fallback: undefined,
  thumb: true,
  lazy: 'lazy',
  rounded: true,
});

const theme = useTheme();
const lazyStyle = {
  backgroundImage: `url(${theme.current.value.dark ? imgCoverDark : imgCoverLight})`,
};

function getThumbSize() {
  if (typeof props.size == 'number') return props.size;
  else if (typeof props.width == 'number' && typeof props.height == 'number') {
    if (props.height > props.width) {
      return props.height;
    } else {
      return props.width;
    }
  } else if (props.thumb) return 256;
  return 0;
}
const thumbSize = getThumbSize();

function getFallbackImage() {
  if (props.fallback) return props.fallback;
  if (
    props.item &&
    'media_type' in props.item &&
    props.item.media_type == MediaType.FOLDER
  )
    return iconFolder;
  if (!props.item) return '';
  if (!props.item.name) return '';
  return getAvatarImage(props.item.name, theme.current.value.dark, thumbSize);
}
const fallbackImage = getFallbackImage();

const imgData = computed(() =>
  props.item
    ? getImageThumbForItem(props.item, ImageType.THUMB, thumbSize) ||
      fallbackImage
    : fallbackImage,
);

// Remove background images
const imageTag = ref(null);
onMounted(() => {
  const callback: IntersectionObserverCallback = (entries) => {
    entries.forEach((entry) => {
      const element = entry.target as HTMLImageElement;
      if (element.complete) {
        element.style.backgroundImage = '';
        observer.disconnect();
      }
    });
  };
  const observer = new IntersectionObserver(callback, {
    root: null,
    threshold: 1.0,
  });
  if (imageTag.value) {
    observer.observe(imageTag.value);
  }
});
</script>

<script lang="ts">
//// utility functions for images

export const getAvatarImage = function (
  name: string,
  dark = false,
  size = 256,
): string {
  // get url to avatar image for a string or sentence
  if (dark)
    return `https://ui-avatars.com/api/?name=${name}&size=${
      size || 256
    }&bold=true&background=1d1d1d&color=383838`;
  else
    return `https://ui-avatars.com/api/?name=${name}&size=${
      size || 256
    }&bold=true&background=a0a0a0&color=cccccc`;
};

export const getMediaItemImage = function (
  mediaItem?: MediaItemType | ItemMapping | QueueItem,
  type: ImageType = ImageType.THUMB,
  includeFileBased = false,
): MediaItemImage | undefined {
  // get imageurl for mediaItem
  if (!mediaItem) return undefined;
  // handle image in queueitem
  if ('image' in mediaItem && mediaItem.image) return mediaItem.image;
  // handle regular image within mediaitem
  if ('metadata' in mediaItem && mediaItem.metadata.images) {
    for (const img of mediaItem.metadata.images) {
      if (!img.remotely_accessible && !includeFileBased) continue;
      if (img.type == type) return img;
    }
  }
  // prefer album image in case of tracks
  if (
    'album' in mediaItem &&
    mediaItem.album &&
    'metadata' in mediaItem.album &&
    mediaItem.album.metadata &&
    mediaItem.album.metadata.images
  ) {
    for (const img of mediaItem.album.metadata.images) {
      if (!img.remotely_accessible && !includeFileBased) continue;
      if (img.type == type) return img;
    }
  }
  // retry with ItemMapping album
  if (
    'album' in mediaItem &&
    mediaItem.album &&
    'image' in mediaItem.album &&
    mediaItem.album.image
  ) {
    if (
      mediaItem.album.image.type == type &&
      (mediaItem.album.image.remotely_accessible || includeFileBased)
    ) {
      return mediaItem.album.image;
    }
  }
  // retry with album artist
  if (
    'artist' in mediaItem &&
    'metadata' in mediaItem.artist &&
    mediaItem.artist.metadata &&
    mediaItem.artist.metadata.images
  ) {
    for (const img of mediaItem.artist.metadata.images) {
      if (!img.remotely_accessible && !includeFileBased) continue;
      if (img.type == type) return img;
    }
  }
  // retry with track artist
  if ('artists' in mediaItem && mediaItem.artists) {
    for (const artist of mediaItem.artists) {
      if ('metadata' in artist && artist.metadata.images) {
        for (const img of artist.metadata.images) {
          if (!img.remotely_accessible && !includeFileBased) continue;
          if (img.type == type) return img;
        }
      }
    }
  }
};

export const getImageThumbForItem = function (
  mediaItem?: MediaItemType | ItemMapping | QueueItem,
  type: ImageType = ImageType.THUMB,
  size?: number,
): string | undefined {
  if (!mediaItem) return;
  // find image in mediaitem
  const img = getMediaItemImage(mediaItem, type, true);
  if (!img || !img.path) return undefined;
  if (!img.remotely_accessible || size) {
    // force imageproxy if image is not remotely accessible or we need a resized thumb
    const encUrl = encodeURIComponent(encodeURIComponent(img.path));
    const checksum =
      'metadata' in mediaItem ? mediaItem.metadata?.cache_checksum : '';
    let imageUrl = `${api.baseUrl}/imageproxy?path=${encUrl}&provider=${img.provider}&checksum=${checksum}`;
    if (size) return imageUrl + `&size=${size}`;
    return imageUrl;
  }
  // else: return image as-is
  return img.path;
};
</script>

<style scoped>
.v-avatar.v-avatar--density-default {
  height: 100% !important;
  width: 100% !important;
}

img {
  min-width: 100%;
  max-width: 100%;
  height: auto;
  display: block;
  background-size: cover;
}

img.rounded {
  border-radius: 4px;
}
</style>
