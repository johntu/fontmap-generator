<template>
  <div class="container mx-auto">
    <h1 class="text-2xl">Font map generator</h1>
    <input type="file" @change="onFile" />
    <br />
    <label for="mapSize">Map size:</label>
    <input id="mapSize" type="number" v-model="opts.mapSize" />
    <label for="fontSize">Font size:</label>
    <input id="fontSize" type="number" v-model="opts.fontSize" />
    <label for="horSpacing">Horizontal spacing:</label>
    <input id="horSpacing" type="number" v-model="opts.horSpacing" />
    <label for="displayBorder">Display border:</label>
    <input id="displayBorder" type="checkbox" v-model="opts.displayBorder" @change="render()" />
    {{opts.displayBorder}}
    <br />
    <textarea class="w-1/2" height="4" v-bind:value="fontMapJson" readonly></textarea>
    <div class="w-1/2 bg-black">
      <img class="w-full" v-bind:src="fontMapSrc" />
    </div>
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
        glyph = font.glyphs.glyphs[characterData[char].index];
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

export default {
  name: "App",
  data: () => {
    return {
      opts: {
        mapSize: 1024,
        fontSize: 90,
        horSpacing: 4,
        displayBorder: true,
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
  },
};
</script>

<style>
</style>
