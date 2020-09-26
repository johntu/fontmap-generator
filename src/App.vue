<template>
  <div class="container mx-auto pt-4">
    <LayoutCard>
      <h1 class="text-2xl mb-1">Font map generator</h1>
      <FileSelect :change="onFile"></FileSelect>
    </LayoutCard>
    <LayoutCard v-if="font">
      <div class="grid xs:grid-cols-2 sm:grid-cols-3 md:grid-cols-4 gap-4">
        <div>
          <label class="block font-bold" for="mapSize">Map size</label>
          <input
            class="w-full bg-gray-100"
            id="mapSize"
            type="number"
            v-model="opts.mapSize"
          />
        </div>
        <div>
          <label class="block font-bold" for="fontSize">Font size</label>
          <input
            class="w-full bg-gray-100"
            id="fontSize"
            type="number"
            v-model="opts.fontSize"
          />
        </div>
        <div>
          <label class="block font-bold" for="horSpacing"
            >Horizontal spacing</label
          >
          <input
            class="w-full bg-gray-100"
            id="horSpacing"
            type="number"
            v-model="opts.horSpacing"
          />
        </div>
        <div>
          <label class="block font-bold" for="displayBorder"
            >Display border</label
          >
          <input
            id="displayBorder"
            type="checkbox"
            v-model="opts.displayBorder"
            @change="render()"
          />
        </div>
      </div>
      <div>
        <div>
          <label class="block font-bold" for="characters">Characters</label>
          <input
            class="w-full bg-gray-100"
            id="characters"
            type="text"
            v-model="opts.characters"
          />
        </div>
      </div>
    </LayoutCard>
    <LayoutCard v-if="font" class="grid md:grid-cols-2 gap-4">
      <div class="flex flex-col">
        <label for="json" class="block font-bold">Font data (JSON)</label>
        <textarea
          class="flex-grow"
          v-bind:value="fontMapJson"
          readonly
        ></textarea>
      </div>
      <div class="bg-black">
        <img class="w-full" v-bind:src="fontMapSrc" />
      </div>
    </LayoutCard>
  </div>
</template>

<script>
import { parse } from "opentype.js";

function loadFont(opts, font) {
  var char,
    glyph,
    scale,
    col,
    w,
    h,
    x,
    y,
    i,
    key,
    xMin = 0,
    xMax = 0,
    characterData = {},
    kerning = {},
    charOfIndex = {};

  scale = opts.fontSize / font.unitsPerEm;

  for (key in font.kerningPairs) {
    var glyphs = key.split(",");

    if (typeof kerning[glyphs[0]] === "undefined") kerning[glyphs[0]] = {};
    kerning[glyphs[0]][glyphs[1]] = font.kerningPairs[key];
  }

  for (i = 0; i < opts.characters.length; i++) {
    char = opts.characters[i];
    charOfIndex[font.charToGlyphIndex(char)] = char;
  }

  for (i = 0; i < opts.characters.length; i++) {
    char = opts.characters[i];
    glyph = font.charToGlyph(char);
    var metrics = glyph.getMetrics();
    characterData[char] = {};
    characterData[char].index = i;
    characterData[char].advanceWidth = glyph.advanceWidth * scale;
    characterData[char].xMin =
      metrics.xMin * scale +
      (metrics.xMin < 0 ? -opts.horSpacing : opts.horSpacing);
    characterData[char].xMax = metrics.xMax * scale - opts.horSpacing;
    characterData[char].yMin = metrics.yMin * scale;
    characterData[char].yMax = metrics.yMax * scale;
    characterData[char].leftSideBearing = metrics.leftSideBearing * scale;
    if (kerning[glyph.index]) {
      characterData[char].kerning = {};
      for (key in kerning[glyph.index]) {
        characterData[char].kerning[charOfIndex[key]] =
          kerning[glyph.index][key] * scale;
      }
    } else {
      characterData[char].kerning = null;
    }
    if (characterData[char].xMin < xMin) xMin = characterData[char].xMin;
    if (characterData[char].xMax > xMax) xMax = characterData[char].xMax;
  }

  var fontData = {
    fontSize: opts.fontSize,
    width: Math.ceil(xMax - xMin),
    height: (font.ascender - font.descender) * scale,
    ascender: font.ascender * scale,
    descender: font.descender * scale,
    characters: characterData,
  };

  // calculate uv coordinates
  col = Math.floor(opts.mapSize / fontData.width);
  w = fontData.width / opts.mapSize;
  h = fontData.height / opts.mapSize;
  for (i = 0; i < opts.characters.length; i++) {
    char = opts.characters[i];
    x = (i - Math.floor(i / col) * col) * w;
    y = 1 - (Math.floor(i / col) + 1) * h;
    characterData[char].uv = [
      [x, y],
      [x + w, y],
      [x + w, y + h],
      [x, y + h],
    ];
  }

  return { fontData, characterData };
}

function renderImage(opts, font, fontData, characterData) {
  var canvas = document.createElement("canvas");
  canvas.width = opts.mapSize;
  canvas.height = opts.mapSize;
  canvas.style.zIndex = 1;

  var ctx = canvas.getContext("2d");
  var char;
  var glyph;
  var path;
  var charWidth = fontData.width;
  var charHeight = fontData.height;
  var mapWidth = Math.floor(opts.mapSize / charWidth);
  var mapHeight = Math.ceil(opts.characters.length / mapWidth);

  // ctx.fillStyle = "#000";
  // ctx.fillRect(0, 0, canvas.width, canvas.height);
  ctx.strokeStyle = "#fff";

  for (var i = 0; i < mapHeight; i++) {
    if (opts.displayBorder) {
      ctx.beginPath();
      ctx.moveTo(0, (i + 1) * charHeight + 0.5);
      ctx.lineTo(canvas.width, (i + 1) * charHeight + 0.5);
      ctx.stroke();
    }
    for (var j = 0; j < mapWidth; j++) {
      if (opts.displayBorder && i === 0) {
        ctx.beginPath();
        ctx.moveTo(j * charWidth + 0.5, 0);
        ctx.lineTo(j * charWidth + 0.5, canvas.height);
        ctx.stroke();
      }
      if (i * mapWidth + j < opts.characters.length) {
        char = opts.characters[i * mapWidth + j];
        glyph = font.charToGlyph(char);
        path = glyph.getPath(
          j * charWidth -
            characterData[char].leftSideBearing +
            Math.abs(characterData[char].xMin),
          (i + 1) * charHeight + fontData.descender,
          opts.fontSize
        );
        path.fill = "white";
        path.draw(ctx);
      }
    }
  }

  return canvas.toDataURL();
}

import FileSelect from "./components/FileSelect.vue";
import LayoutCard from "./components/LayoutCard.vue";

import robotoFont from "./assets/Roboto-Bold.ttf";

var debug = false;

export default {
  name: "App",
  components: {
    FileSelect,
    LayoutCard,
  },
  data: () => {
    return {
      opts: {
        mapSize: 1024,
        fontSize: 90,
        horSpacing: 4,
        displayBorder: false,
        characters:
          "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789.,:;!?\"'@#$%&+-*/\\()[]{}]",
      },
      font: null,
      fontData: null,
      characterData: null,
      fontMapJson: null,
      fontMapSrc: null,
    };
  },
  mounted: function () {
    if (debug) {
      var rawFile = new XMLHttpRequest();
      rawFile.open("GET", robotoFont, true);
      rawFile.responseType = "arraybuffer";
      rawFile.onreadystatechange = () => {
        if (rawFile.readyState === 4) {
          if (rawFile.status === 200 || rawFile.status == 0) {
            this.font = parse(rawFile.response);

            this.render();
          }
        }
      };
      rawFile.send(null);
    }
  },
  methods: {
    onFile(event) {
      var file = event.target.files[0];

      file.arrayBuffer().then((buffer) => {
        this.font = parse(buffer);

        this.render();
      });
    },
    render() {
      const { fontData, characterData } = loadFont(this.opts, this.font);

      this.fontData = fontData;
      this.characterData = characterData;

      this.fontMapJson = JSON.stringify(this.fontData);
      this.fontMapSrc = renderImage(
        this.opts,
        this.font,
        this.fontData,
        this.characterData
      );
    },
  },
  watch: {
    "opts.mapSize": function () {
      this.render();
    },
    "opts.fontSize": function () {
      this.render();
    },
    "opts.horSpacing": function () {
      this.render();
    },
    "opts.displayBorder": function () {
      this.render();
    },
    "opts.characters": function () {
      this.render();
    },
  },
};
</script>

<style>
</style>
