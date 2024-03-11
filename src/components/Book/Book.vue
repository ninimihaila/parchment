<template>
  <input type="file" id="file-input" hidden>
  <div v-show="!hidePicker" id="drop-target" class="filter">
      <div>
          <svg class="icon empty-state-icon" width="72" height="72" aria-hidden="true">
              <path d="M36 18s-6-6-12-6-15 6-15 6v42s9-6 15-6 12 6 12 6c4-4 8-6 12-6s12 2 15 6V18c-6-4-12-6-15-6-4 0-8 2-12 6m0 0v42"/>
          </svg>
          <h1>Drop a book here!</h1>
          <p>Or <button id="file-button">choose a file</button> to open it.</p>
      </div>
  </div>
  <div id="dimming-overlay" aria-hidden="true"></div>
  <div id="side-bar">
      <div id="side-bar-header">
          <img id="side-bar-cover">
          <div>
              <h1 id="side-bar-title"></h1>
              <p id="side-bar-author"></p>
          </div>
      </div>
      <div id="toc-view"></div>
  </div>
  <div id="header-bar" class="toolbar">
      <button id="side-bar-button" aria-label="Show sidebar">
          <svg class="icon" width="24" height="24" aria-hidden="true">
              <path d="M 4 6 h 16 M 4 12 h 16 M 4 18 h 16"/>
          </svg>
      </button>
      <div id="menu-button" class="menu-container">
          <button aria-label="Show settings" aria-haspopup="true">
              <svg class="icon" width="24" height="24" aria-hidden="true">
                  <path d="M5 12.7a7 7 0 0 1 0-1.4l-1.8-2 2-3.5 2.7.5a7 7 0 0 1 1.2-.7L10 3h4l.9 2.6 1.2.7 2.7-.5 2 3.4-1.8 2a7 7 0 0 1 0 1.5l1.8 2-2 3.5-2.7-.5a7 7 0 0 1-1.2.7L14 21h-4l-.9-2.6a7 7 0 0 1-1.2-.7l-2.7.5-2-3.4 1.8-2Z"/>
                  <circle cx="12" cy="12" r="3"/>
              </svg>
          </button>
      </div>
  </div>
  <div id="nav-bar" class="toolbar">
      <button id="left-button" aria-label="Go left">
          <svg class="icon" width="24" height="24" aria-hidden="true">
              <path d="M 15 6 L 9 12 L 15 18"/>
          </svg>
      </button>
      <input id="progress-slider" type="range" min="0" max="1" step="any" list="tick-marks">
      <datalist id="tick-marks"></datalist>
      <button id="right-button" aria-label="Go right">
          <svg class="icon" width="24" height="24" aria-hidden="true">
              <path d="M 9 6 L 15 12 L 9 18"/>
          </svg>
      </button>
  </div>
</template>

<script setup lang="ts">
// import "./foliate/vendor/pdfjs/pdf.js";
import "./foliate/view.js";
import { createTOCView } from './foliate/ui/tree.js'
import { createMenu } from './foliate/ui/menu.js'
import { Overlayer } from './foliate/overlayer.js'

import { onMounted, ref } from "vue";

const hidePicker = ref(false);

onMounted(() => {
const isZip = async file => {
    const arr = new Uint8Array(await file.slice(0, 4).arrayBuffer())
    return arr[0] === 0x50 && arr[1] === 0x4b && arr[2] === 0x03 && arr[3] === 0x04
}

const isPDF = async file => {
    const arr = new Uint8Array(await file.slice(0, 5).arrayBuffer())
    return arr[0] === 0x25
        && arr[1] === 0x50 && arr[2] === 0x44 && arr[3] === 0x46
        && arr[4] === 0x2d
}

const makeZipLoader = async file => {
    const { configure, ZipReader, BlobReader, TextWriter, BlobWriter } =
        await import('./foliate/vendor/zip.js')
    configure({ useWebWorkers: false })
    const reader = new ZipReader(new BlobReader(file))
    const entries = await reader.getEntries()
    const map = new Map(entries.map(entry => [entry.filename, entry]))
    const load = f => (name, ...args) =>
        map.has(name) ? f(map.get(name), ...args) : null
    const loadText = load(entry => entry.getData(new TextWriter()))
    const loadBlob = load((entry, type) => entry.getData(new BlobWriter(type)))
    const getSize = name => map.get(name)?.uncompressedSize ?? 0
    return { entries, loadText, loadBlob, getSize }
}

const getFileEntries = async entry => entry.isFile ? entry
    : (await Promise.all(Array.from(
        await new Promise((resolve, reject) => entry.createReader()
            .readEntries(entries => resolve(entries), error => reject(error))),
        getFileEntries))).flat()

const makeDirectoryLoader = async entry => {
    const entries = await getFileEntries(entry)
    const files = await Promise.all(
        entries.map(entry => new Promise((resolve, reject) =>
            entry.file(file => resolve([file, entry.fullPath]),
                error => reject(error)))))
    const map = new Map(files.map(([file, path]) =>
        [path.replace(entry.fullPath + '/', ''), file]))
    const decoder = new TextDecoder()
    const decode = x => x ? decoder.decode(x) : null
    const getBuffer = name => map.get(name)?.arrayBuffer() ?? null
    const loadText = async name => decode(await getBuffer(name))
    const loadBlob = name => map.get(name)
    const getSize = name => map.get(name)?.size ?? 0
    return { loadText, loadBlob, getSize }
}

const isCBZ = ({ name, type }) =>
    type === 'application/vnd.comicbook+zip' || name.endsWith('.cbz')

const isFB2 = ({ name, type }) =>
    type === 'application/x-fictionbook+xml' || name.endsWith('.fb2')

const isFBZ = ({ name, type }) =>
    type === 'application/x-zip-compressed-fb2'
    || name.endsWith('.fb2.zip') || name.endsWith('.fbz')

const getView = async file => {
    let book
    if (file.isDirectory) {
        const loader = await makeDirectoryLoader(file)
        const { EPUB } = await import('./foliate/epub.js')
        book = await new EPUB(loader).init()
    }
    else if (!file.size) throw new Error('File not found')
    else if (await isZip(file)) {
        const loader = await makeZipLoader(file)
        if (isCBZ(file)) {
            const { makeComicBook } = await import('./foliate/comic-book.js')
            book = makeComicBook(loader, file)
        } else if (isFBZ(file)) {
            const { makeFB2 } = await import('./foliate/fb2.js')
            const { entries } = loader
            const entry = entries.find(entry => entry.filename.endsWith('.fb2'))
            const blob = await loader.loadBlob((entry ?? entries[0]).filename)
            book = await makeFB2(blob)
        } else {
            const { EPUB } = await import('./foliate/epub.js')
            book = await new EPUB(loader).init()
        }
    }
    else if (await isPDF(file)) {
        const { makePDF } = await import('./foliate/pdf.js')
        book = await makePDF(file)
    }
    else {
        const { isMOBI, MOBI } = await import('./foliate/mobi.js')
        if (await isMOBI(file)) {
            const fflate = await import('./foliate/vendor/fflate.js')
            book = await new MOBI({ unzlib: fflate.unzlibSync }).open(file)
        } else if (isFB2(file)) {
            const { makeFB2 } = await import('./foliate/fb2.js')
            book = await makeFB2(file)
        }
    }
    if (!book) throw new Error('File type not supported')

    const view = document.createElement('foliate-view')
    document.body.append(view)
    await view.open(book)
    return view
}

const getCSS = ({ spacing, justify, hyphenate }) => `
    @namespace epub "http://www.idpf.org/2007/ops";
    html {
        color-scheme: light dark;
    }
    /* https://github.com/whatwg/html/issues/5426 */
    @media (prefers-color-scheme: dark) {
        a:link {
            color: lightblue;
        }
    }
    p, li, blockquote, dd {
        line-height: ${spacing};
        text-align: ${justify ? 'justify' : 'start'};
        -webkit-hyphens: ${hyphenate ? 'auto' : 'manual'};
        hyphens: ${hyphenate ? 'auto' : 'manual'};
        -webkit-hyphenate-limit-before: 3;
        -webkit-hyphenate-limit-after: 2;
        -webkit-hyphenate-limit-lines: 2;
        hanging-punctuation: allow-end last;
        widows: 2;
    }
    /* prevent the above from overriding the align attribute */
    [align="left"] { text-align: left; }
    [align="right"] { text-align: right; }
    [align="center"] { text-align: center; }
    [align="justify"] { text-align: justify; }

    pre {
        white-space: pre-wrap !important;
    }
    aside[epub|type~="endnote"],
    aside[epub|type~="footnote"],
    aside[epub|type~="note"],
    aside[epub|type~="rearnote"] {
        display: none;
    }
`

const $ = document.querySelector.bind(document)

const locales = 'en'
const percentFormat = new Intl.NumberFormat(locales, { style: 'percent' })

class Reader {
    #tocView
    style = {
        spacing: 1.4,
        justify: true,
        hyphenate: true,
    }
    annotations = new Map()
    annotationsByValue = new Map()
    closeSideBar() {
        $('#dimming-overlay').classList.remove('show')
        $('#side-bar').classList.remove('show')
    }
    constructor() {
        $('#side-bar-button').addEventListener('click', () => {
            $('#dimming-overlay').classList.add('show')
            $('#side-bar').classList.add('show')
        })
        $('#dimming-overlay').addEventListener('click', () => this.closeSideBar())

        const menu = createMenu([
            {
                name: 'layout',
                label: 'Layout',
                type: 'radio',
                items: [
                    ['Paginated', 'paginated'],
                    ['Scrolled', 'scrolled'],
                ],
                onclick: value => {
                    this.view?.renderer.setAttribute('flow', value)
                },
            },
        ])
        menu.element.classList.add('menu')

        $('#menu-button').append(menu.element)
        $('#menu-button > button').addEventListener('click', () =>
            menu.element.classList.toggle('show'))
        menu.groups.layout.select('paginated')
    }
    async open(file) {
        this.view = await getView(file)
        this.view.addEventListener('load', this.#onLoad.bind(this))
        this.view.addEventListener('relocate', this.#onRelocate.bind(this))

        const { book } = this.view
        this.view.renderer.setStyles?.(getCSS(this.style))
        this.view.renderer.next()

        $('#header-bar').style.visibility = 'visible'
        $('#nav-bar').style.visibility = 'visible'
        $('#left-button').addEventListener('click', () => this.view.goLeft())
        $('#right-button').addEventListener('click', () => this.view.goRight())

        const slider = $('#progress-slider')
        slider.dir = book.dir
        slider.addEventListener('input', e =>
            this.view.goToFraction(parseFloat(e.target.value)))
        for (const fraction of this.view.getSectionFractions()) {
            const option = document.createElement('option')
            option.value = fraction
            $('#tick-marks').append(option)
        }

        document.addEventListener('keydown', this.#handleKeydown.bind(this))

        const title = book.metadata?.title ?? 'Untitled Book'
        document.title = title
        $('#side-bar-title').innerText = title
        const author = book.metadata?.author
        $('#side-bar-author').innerText = typeof author === 'string' ? author
            : author
                ?.map(author => typeof author === 'string' ? author : author.name)
                ?.join(', ')
                ?? ''
        Promise.resolve(book.getCover?.())?.then(blob =>
            blob ? $('#side-bar-cover').src = URL.createObjectURL(blob) : null)

        const toc = book.toc
        if (toc) {
            this.#tocView = createTOCView(toc, href => {
                this.view.goTo(href).catch(e => console.error(e))
                this.closeSideBar()
            })
            $('#toc-view').append(this.#tocView.element)
        }

        // load and show highlights embedded in the file by Calibre
        const bookmarks = await book.getCalibreBookmarks?.()
        if (bookmarks) {
            const { fromCalibreHighlight } = await import('./foliate/epubcfi.js')
            for (const obj of bookmarks) {
                if (obj.type === 'highlight') {
                    const value = fromCalibreHighlight(obj)
                    const color = obj.style.which
                    const note = obj.notes
                    const annotation = { value, color, note }
                    const list = this.annotations.get(obj.spine_index)
                    if (list) list.push(annotation)
                    else this.annotations.set(obj.spine_index, [annotation])
                    this.annotationsByValue.set(value, annotation)
                }
            }
            this.view.addEventListener('create-overlay', e => {
                const { index } = e.detail
                const list = this.annotations.get(index)
                if (list) for (const annotation of list)
                    this.view.addAnnotation(annotation)
            })
            this.view.addEventListener('draw-annotation', e => {
                const { draw, annotation } = e.detail
                const { color } = annotation
                draw(Overlayer.highlight, { color })
            })
            this.view.addEventListener('show-annotation', e => {
                const annotation = this.annotationsByValue.get(e.detail.value)
                if (annotation.note) alert(annotation.note)
            })
        }
    }
    #handleKeydown(event) {
        const k = event.key
        if (k === 'ArrowLeft' || k === 'h') this.view.goLeft()
        else if(k === 'ArrowRight' || k === 'l') this.view.goRight()
    }
    #onLoad({ detail: { doc } }) {
        doc.addEventListener('keydown', this.#handleKeydown.bind(this))
    }
    #onRelocate({ detail }) {
        const { fraction, location, tocItem, pageItem } = detail
        const percent = percentFormat.format(fraction)
        const loc = pageItem
            ? `Page ${pageItem.label}`
            : `Loc ${location.current}`
        const slider = $('#progress-slider')
        slider.style.visibility = 'visible'
        slider.value = fraction
        slider.title = `${percent} · ${loc}`
        if (tocItem?.href) this.#tocView?.setCurrentHref?.(tocItem.href)
    }
}

const open = async file => {
    hidePicker.value = true;
    const reader = new Reader()
    globalThis.reader = reader
    await reader.open(file)
}

const dragOverHandler = e => e.preventDefault()
const dropHandler = e => {
    e.preventDefault()
    const item = Array.from(e.dataTransfer.items)
        .find(item => item.kind === 'file')
    if (item) {
        const entry = item.webkitGetAsEntry()
        open(entry.isFile ? item.getAsFile() : entry).catch(e => console.error(e))
    }
}
const dropTarget = $('#drop-target')
dropTarget.addEventListener('drop', dropHandler)
dropTarget.addEventListener('dragover', dragOverHandler)

$('#file-input').addEventListener('change', e =>
    open(e.target.files[0]).catch(e => console.error(e)))
$('#file-button').addEventListener('click', () => $('#file-input').click())

const params = new URLSearchParams(location.search)
const url = params.get('url')
if (url) fetch(url)
    .then(res => res.blob())
    .then(blob => open(new File([blob], new URL(url).pathname)))
    .catch(e => console.error(e))
else dropTarget.style.visibility = 'visible'

})
</script>

<style>
:root {
    --active-bg: rgba(0, 0, 0, .05);
}
@supports (color-scheme: light dark) {
    @media (prefers-color-scheme: dark) {
        :root {
            --active-bg: rgba(255, 255, 255, .1);
        }
    }
}
html {
    height: 100%;
}
body {
    margin: 0 auto;
    height: 100%;
    font: menu;
    font-family: system-ui, sans-serif;
}
#drop-target {
    height: 100vh;
    display: flex;
    align-items: center;
    justify-content: center;
    text-align: center;
    visibility: hidden;
}
#drop-target h1 {
    font-weight: 900;
}
#file-button {
    font: inherit;
    background: none;
    border: 0;
    padding: 0;
    text-decoration: underline;
    cursor: pointer;
}
.icon {
    display: block;
    fill: none;
    stroke: currentcolor;
    stroke-width: 2px;
}
.empty-state-icon {
    margin: auto;
}
.toolbar {
    box-sizing: border-box;
    position: absolute;
    z-index: 1;
    display: flex;
    align-items: center;
    justify-content: space-between;
    width: 100%;
    height: 48px;
    padding: 6px;
    transition: opacity 250ms ease;
    visibility: hidden;
}
.toolbar button {
    padding: 3px;
    border-radius: 6px;
    background: none;
    border: 0;
    color: GrayText;
}
.toolbar button:hover {
    background: rgba(0, 0, 0, .1);
    color: currentcolor;
}
#header-bar {
    top: 0;
}
#nav-bar {
    bottom: 0;
}
#progress-slider {
    flex-grow: 1;
    margin: 0 12px;
    visibility: hidden;
}
#side-bar {
    visibility: hidden;
    box-sizing: border-box;
    position: absolute;
    z-index: 2;
    top: 0;
    left: 0;
    height: 100%;
    width: 320px;
    transform: translateX(-320px);
    display: flex;
    flex-direction: column;
    background: Canvas;
    color: CanvasText;
    box-shadow: 0 0 0 1px rgba(0, 0, 0, .2), 0 0 40px rgba(0, 0, 0, .2);
    transition: visibility 0s linear 300ms, transform 300ms ease;
}
#side-bar.show {
    visibility: visible;
    transform: translateX(0);
    transition-delay: 0s;
}
#dimming-overlay {
    visibility: hidden;
    position: fixed;
    z-index: 2;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: rgba(0, 0, 0, .2);
    opacity: 0;
    transition: visibility 0s linear 300ms, opacity 300ms ease;
}
#dimming-overlay.show {
    visibility: visible;
    opacity: 1;
    transition-delay: 0s;
}
#side-bar-header {
    padding: 1rem;
    display: flex;
    border-bottom: 1px solid rgba(0, 0, 0, .1);
    align-items: center;
}
#side-bar-cover {
    height: 10vh;
    min-height: 60px;
    max-height: 180px;
    border-radius: 3px;
    border: 0;
    background: lightgray;
    box-shadow: 0 0 1px rgba(0, 0, 0, .1), 0 0 16px rgba(0, 0, 0, .1);
    margin-inline-end: 1rem;
}
#side-bar-cover:not([src]) {
    display: none;
}
#side-bar-title {
    margin: .5rem 0;
    font-size: inherit;
}
#side-bar-author {
    margin: .5rem 0;
    font-size: small;
    color: GrayText;
}
#toc-view {
    padding: .5rem;
    overflow-y: scroll;
}
#toc-view li, #toc-view ol {
    margin: 0;
    padding: 0;
    list-style: none;
}
#toc-view a, #toc-view span {
    display: block;
    border-radius: 6px;
    padding: 8px;
    margin: 2px 0;
}
#toc-view a {
    color: CanvasText;
    text-decoration: none;
}
#toc-view a:hover {
    background: var(--active-bg);
}
#toc-view span {
    color: GrayText;
}
#toc-view svg {
    margin-inline-start: -24px;
    padding-inline-start: 5px;
    padding-inline-end: 6px;
    fill: CanvasText;
    cursor: default;
    transition: transform .2s ease;
    opacity: .5;
}
#toc-view svg:hover {
    opacity: 1;
}
#toc-view [aria-current] {
    font-weight: bold;
    background: var(--active-bg);
}
#toc-view [aria-expanded="false"] svg {
    transform: rotate(-90deg);
}
#toc-view [aria-expanded="false"] + [role="group"] {
    display: none;
}
.menu-container {
    position: relative;
}
.menu, .menu ul {
    list-style: none;
    padding: 0;
    margin: 0;
}
.menu {
    visibility: hidden;
    position: absolute;
    right: 0;
    background: Canvas;
    color: CanvasText;
    border-radius: 6px;
    box-shadow: 0 0 0 1px rgba(0, 0, 0, .2), 0 0 16px rgba(0, 0, 0, .1);
    padding: 6px;
    cursor: default;
}
.menu.show {
    visibility: visible;
}
.menu li {
    padding: 6px 12px;
    padding-left: 24px;
    border-radius: 6px;
}
.menu li:hover {
    background: var(--active-bg);
}
.menu li[aria-checked="true"] {
    background-position: center left;
    background-repeat: no-repeat;
    background-image: url('data:image/svg+xml;utf8,%3Csvg%20xmlns%3D%22http%3A%2F%2Fwww.w3.org%2F2000%2Fsvg%22%20width%3D%2224%22%20height%3D%2224%22%3E%3Ccircle%20cx%3D%2212%22%20cy%3D%2212%22%20r%3D%223%22%2F%3E%3C%2Fsvg%3E');
}
.popover {
    background: Canvas;
    color: CanvasText;
    border-radius: 6px;
    box-shadow: 0 0 0 1px rgba(0, 0, 0, .2), 0 0 16px rgba(0, 0, 0, .1), 0 0 32px rgba(0, 0, 0, .1);
}
.popover-arrow-down {
    fill: Canvas;
    filter: drop-shadow(0 -1px 0 rgba(0, 0, 0, .2));
}
.popover-arrow-up {
    fill: Canvas;
    filter: drop-shadow(0 1px 0 rgba(0, 0, 0, .2));
}
</style>
