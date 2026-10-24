<template>
  <div id="epub-frame" class="w-full">
    <div id="viewer" class="h-full w-full" :style="pullViewerStyle"></div>

    <div v-if="pullEdge && pullCanNavigate" class="fixed left-0 w-full flex flex-col items-center pointer-events-none" :style="pullIndicatorStyle">
      <p class="text-xs bg-black/60 text-white px-3 py-1 rounded">{{ pullArmed ? $strings.LabelHoldToContinue : pullEdge === 'end' ? $strings.LabelPullForNextChapter : $strings.LabelPullForPreviousChapter }}</p>
      <div v-if="pullArmed" class="mt-1 w-32 h-1 bg-white/30 rounded overflow-hidden">
        <div class="pull-progress h-full bg-white" />
      </div>
    </div>

    <div class="fixed left-0 h-8 w-full px-4 flex items-center" :class="isLightTheme ? 'bg-white text-black' : isDarkTheme ? 'bg-[#232323] text-white/80' : 'bg-black text-white/80'" :style="{ bottom: isPlayerOpen ? '120px' : '0px' }">
      <p v-if="totalLocations" class="text-xs text-slate-600">Location {{ currentLocationNum }} of {{ totalLocations }}</p>
      <div class="flex-grow" />
      <p class="text-xs">{{ progress }}%</p>
    </div>
  </div>
</template>

<script>
import ePub from 'epubjs'

// Used when the epub does not set rendition:flow
const DEFAULT_FLOW = 'scrolled'

// Overscroll at the end or start of a chapter in scrolled mode
const PULL_ARM_DISTANCE = 140 // px pulled past the edge before the hold starts
const PULL_HOLD_MS = 500 // keep holding this long to change chapter. The pull-progress animation in the style block uses the same duration
const PULL_RESISTANCE = 0.5 // how far the content moves per px of pull
const PULL_MAX_OFFSET = 120 // max px the content moves

export default {
  props: {
    url: String,
    libraryItem: {
      type: Object,
      default: () => {}
    },
    isLocal: Boolean,
    keepProgress: Boolean
  },
  data() {
    return {
      /** @type {ePub.Book} */
      book: null,
      /** @type {ePub.Rendition} */
      rendition: null,
      currentFlow: null,
      renditionWidth: null,
      renditionHeightOffset: null,
      pullEdge: null,
      pullStartY: null,
      pullTouchStartY: null,
      pullTriggered: false,
      pullCanNavigate: false,
      pullActive: false,
      pullArmed: false,
      pullOffset: 0,
      pullHoldTimeout: null,
      progress: 0,
      totalLocations: 0,
      currentLocationNum: 0,
      currentLocationCfi: null,
      inittingDisplay: true,
      isRefreshingUI: false,
      ereaderSettings: {
        theme: 'dark',
        font: 'serif',
        fontScale: 100,
        lineSpacing: 115,
        textStroke: 0,
        flow: null
      }
    }
  },
  watch: {
    isPlayerOpen() {
      this.refreshUI()
    }
  },
  computed: {
    /** @returns {string} */
    libraryItemId() {
      return this.libraryItem?.id
    },
    localLibraryItem() {
      if (this.isLocal) return this.libraryItem
      return this.libraryItem.localLibraryItem || null
    },
    localLibraryItemId() {
      return this.localLibraryItem?.id
    },
    serverLibraryItemId() {
      if (!this.isLocal) return this.libraryItem.id
      // Check if local library item is connected to the current server
      if (!this.libraryItem.serverAddress || !this.libraryItem.libraryItemId) return null
      if (this.$store.getters['user/getServerAddress'] === this.libraryItem.serverAddress) {
        return this.libraryItem.libraryItemId
      }
      return null
    },
    isPlayerOpen() {
      return this.$store.getters['getIsPlayerOpen']
    },
    readerHeightOffset() {
      return this.isPlayerOpen ? 204 : 64
    },
    /** @returns {Array<ePub.NavItem>} */
    chapters() {
      return this.book?.navigation?.toc || []
    },
    userItemProgress() {
      if (this.isLocal) return this.localItemProgress
      return this.serverItemProgress
    },
    localItemProgress() {
      return this.$store.getters['globals/getLocalMediaProgressById'](this.localLibraryItemId)
    },
    serverItemProgress() {
      return this.$store.getters['user/getUserMediaProgress'](this.serverLibraryItemId)
    },
    localStorageLocationsKey() {
      return `ebookLocations-${this.libraryItemId}`
    },
    savedEbookLocation() {
      if (!this.keepProgress) return null
      if (!this.userItemProgress?.ebookLocation) return null
      // Validate ebookLocation is an epubcfi
      if (!String(this.userItemProgress.ebookLocation).startsWith('epubcfi')) return null
      return this.userItemProgress.ebookLocation
    },
    isLightTheme() {
      return this.ereaderSettings.theme === 'light'
    },
    isDarkTheme() {
      return this.ereaderSettings.theme === 'dark'
    },
    pullViewerStyle() {
      return {
        transform: `translateY(${this.pullOffset}px)`,
        // Springs back when the finger lifts, follows the finger while pulling
        transition: this.pullActive ? 'none' : 'transform 0.25s ease-out'
      }
    },
    pullIndicatorStyle() {
      if (this.pullEdge === 'end') return { bottom: this.isPlayerOpen ? '152px' : '40px' }
      return { top: '40px' }
    },
    themeRules() {
      const isDark = this.ereaderSettings.theme === 'dark'
      const isBlack = this.ereaderSettings.theme === 'black'
      const fontColor = isDark ? '#fff' : isBlack ? '#fff' : '#000'
      const backgroundColor = isDark ? 'rgb(35 35 35)' : isBlack ? 'rgb(0 0 0)' : 'rgb(255, 255, 255)'

      return {
        '*': {
          color: `${fontColor}!important`,
          'background-color': `${backgroundColor}!important`,
          'line-height': this.ereaderSettings.lineSpacing + '%!important',
          '-webkit-text-stroke': this.ereaderSettings.textStroke / 100 + 'px ' + fontColor + '!important'
        },
        a: {
          color: `${fontColor}!important`
        }
      }
    }
  },
  methods: {
    updateSettings(settings) {
      this.ereaderSettings = settings

      if (!this.rendition) return

      if (settings.flow && settings.flow !== this.currentFlow) {
        this.setFlow(settings.flow)
        return
      }

      this.applyTheme()
      this.applyDisplaySettings()
    },
    goToChapter(href) {
      return this.rendition?.display(href)
    },
    prev() {
      if (this.rendition) {
        this.rendition.prev()
      }
    },
    next() {
      if (this.rendition) {
        this.rendition.next()
      }
    },
    /**
     * @param {object} payload
     * @param {string} payload.ebookLocation - CFI of the current location
     * @param {string} payload.ebookProgress - eBook Progress Percentage
     */
    async updateProgress(payload) {
      if (!this.keepProgress) return

      // Update local item
      if (this.localLibraryItemId) {
        const localPayload = {
          localLibraryItemId: this.localLibraryItemId,
          ...payload
        }
        const localResponse = await this.$db.updateLocalEbookProgress(localPayload)
        if (localResponse.localMediaProgress) {
          this.$store.commit('globals/updateLocalMediaProgress', localResponse.localMediaProgress)
        }
      }

      // Update server item
      if (this.serverLibraryItemId) {
        this.$nativeHttp.patch(`/api/me/progress/${this.serverLibraryItemId}`, payload).catch((error) => {
          console.error('EpubReader.updateProgress failed:', error)
        })
      }
    },
    getAllEbookLocationData() {
      const locations = []
      let totalSize = 0 // Total in bytes

      for (const key in localStorage) {
        if (!localStorage.hasOwnProperty(key) || !key.startsWith('ebookLocations-')) {
          continue
        }

        try {
          const ebookLocations = JSON.parse(localStorage[key])
          if (!ebookLocations.locations) throw new Error('Invalid locations object')

          ebookLocations.key = key
          ebookLocations.size = (localStorage[key].length + key.length) * 2
          locations.push(ebookLocations)
          totalSize += ebookLocations.size
        } catch (error) {
          console.error('Failed to parse ebook locations', key, error)
          localStorage.removeItem(key)
        }
      }

      // Sort by oldest lastAccessed first
      locations.sort((a, b) => a.lastAccessed - b.lastAccessed)

      return {
        locations,
        totalSize
      }
    },
    /** @param {string} locationString */
    checkSaveLocations(locationString) {
      const maxSizeInBytes = 3000000 // Allow epub locations to take up to 3MB of space
      const newLocationsSize = JSON.stringify({ lastAccessed: Date.now(), locations: locationString }).length * 2

      // Too large overall
      if (newLocationsSize > maxSizeInBytes) {
        console.error('Epub locations are too large to store. Size =', newLocationsSize)
        return
      }

      const ebookLocationsData = this.getAllEbookLocationData()

      let availableSpace = maxSizeInBytes - ebookLocationsData.totalSize

      // Remove epub locations until there is room for locations
      while (availableSpace < newLocationsSize && ebookLocationsData.locations.length) {
        const oldestLocation = ebookLocationsData.locations.shift()
        console.log(`Removing cached locations for epub "${oldestLocation.key}" taking up ${oldestLocation.size} bytes`)
        availableSpace += oldestLocation.size
        localStorage.removeItem(oldestLocation.key)
      }

      console.log(`Cacheing epub locations with key "${this.localStorageLocationsKey}" taking up ${newLocationsSize} bytes`)
      this.saveLocations(locationString)
    },
    /** @param {string} locationString */
    saveLocations(locationString) {
      localStorage.setItem(
        this.localStorageLocationsKey,
        JSON.stringify({
          lastAccessed: Date.now(),
          locations: locationString
        })
      )
    },
    loadLocations() {
      const locationsObjString = localStorage.getItem(this.localStorageLocationsKey)
      if (!locationsObjString) return null

      const locationsObject = JSON.parse(locationsObjString)

      // Remove invalid location objects
      if (!locationsObject.locations) {
        console.error('Invalid epub locations stored', this.localStorageLocationsKey)
        localStorage.removeItem(this.localStorageLocationsKey)
        return null
      }

      // Update lastAccessed
      this.saveLocations(locationsObject.locations)

      return locationsObject.locations
    },
    /** @param {string} location - CFI of the new location */
    relocated(location) {
      console.log(`[EpubReader] relocated ${location.start.cfi}`)
      if (this.inittingDisplay) {
        console.log(`[EpubReader] relocated but initting display ${location.start.cfi}`)
        return
      }
      this.currentLocationNum = location.start.location

      if (this.currentLocationCfi === location.start.cfi) {
        console.log(`[EpubReader] location already saved`, location.start.cfi)
        return
      }

      console.log(`[EpubReader] Saving new location ${location.start.cfi}`)
      this.currentLocationCfi = location.start.cfi

      if (location.end.percentage) {
        this.updateProgress({
          ebookLocation: location.start.cfi,
          ebookProgress: location.end.percentage
        })
        this.progress = Math.round(location.end.percentage * 100)
      } else {
        this.updateProgress({
          ebookLocation: location.start.cfi
        })
      }
    },
    initEpub() {
      this.progress = Math.round((this.userItemProgress?.ebookProgress || 0) * 100)

      /** @type {EpubReader} */
      const reader = this

      // Use axios to make request because we have token refresh logic in interceptor
      const customRequest = async (url) => {
        try {
          return this.$axios.$get(url, {
            responseType: 'arraybuffer'
          })
        } catch (error) {
          console.error('EpubReader.initEpub customRequest failed:', error)
          throw error
        }
      }

      console.log('[EpubReader] initEpub', reader.url)
      /** @type {ePub.Book} */
      reader.book = new ePub(reader.url, {
        width: window.innerWidth,
        height: window.innerHeight - this.readerHeightOffset,
        openAs: 'epub',
        requestMethod: this.isLocal ? null : customRequest
      })

      reader.book.ready.then(() => {
        console.log('%c [EpubReader] Book ready', 'color:cyan;')

        // The flow decides the rendition's manager, so the rendition can only be created once the book's metadata is known
        const epubFlow = this.getEpubFlow()
        this.$emit('flow-resolved', epubFlow)
        this.createRendition(epubFlow)

        let displayCfi = reader.book.locations.start
        if (this.savedEbookLocation && reader.book.spine.get(this.savedEbookLocation)) {
          displayCfi = this.savedEbookLocation
        }

        // load ebook cfi locations
        const savedLocations = this.loadLocations()
        if (savedLocations) {
          reader.book.locations.load(savedLocations)
          this.totalLocations = reader.book.locations.length()
        } else {
          reader.book.locations.generate(100).then(() => {
            this.totalLocations = reader.book.locations.length()
            this.currentLocationNum = reader.rendition.currentLocation()?.start.location || 0
            this.checkSaveLocations(reader.book.locations.save())
          })
        }

        // TODO: To get the correct page need to render twice. On book ready and after first display. Figure out why
        console.log(`[EpubReader] Displaying cfi ${displayCfi}`)
        this.currentLocationCfi = displayCfi
        reader.rendition.display(displayCfi).then(() => {
          reader.rendition.display(displayCfi).then(() => {
            this.inittingDisplay = false
          })
        })
      })
    },
    /** @returns {string} flow set by the epub (rendition:flow), or the default if it is missing or unknown */
    getEpubFlow() {
      const flow = this.book?.packaging?.metadata?.flow
      if (flow === 'paginated') return 'paginated'
      // scrolled-doc and scrolled-continuous are both shown as scrolled
      if (flow === 'scrolled-doc' || flow === 'scrolled-continuous') return 'scrolled'
      return DEFAULT_FLOW
    },
    /** @param {string} flow */
    createRendition(flow) {
      /** @type {ePub.Rendition} */
      const rendition = this.book.renderTo('viewer', {
        width: window.innerWidth,
        height: window.innerHeight - this.readerHeightOffset,
        // Snapping only applies to paginated
        snap: flow === 'paginated',
        manager: flow === 'scrolled' ? 'default' : 'continuous',
        flow: flow === 'scrolled' ? 'scrolled-doc' : flow
      })
      this.rendition = rendition
      this.currentFlow = flow
      this.renditionWidth = window.innerWidth
      this.renditionHeightOffset = this.readerHeightOffset

      rendition.on('displayed', async () => {
        console.log('%c [EpubReader] Rendition displayed', 'color:blue;')

        // Overriding the needsSnap function in epubjs `snap.js` to fix a bug with scrollLeft being a decimal
        const snapper = rendition.manager.snapper
        if (snapper) {
          snapper.needsSnap = function () {
            let left = Math.round(this.scrollLeft)
            let snapWidth = this.layout.pageWidth * this.layout.divisor
            return left % snapWidth !== 0
          }
        }
      })

      rendition.on('rendered', (section, view) => {
        this.applyTheme()
        console.log('%c [EpubReader] Rendition rendered', 'color:red;', section, view)
      })

      // set up event listeners
      rendition.on('relocated', this.relocated)

      rendition.on('displayError', (err) => {
        console.log('[EpubReader] Display error', err)
      })

      rendition.on('touchstart', (event) => {
        this.resetScrolledPull()
        this.pullTouchStartY = event.touches[0].screenY
        this.$emit('touchstart', event)
      })
      rendition.on('touchmove', (event) => {
        this.handleScrolledPull(event)
      })
      rendition.on('touchend', (event) => {
        this.resetScrolledPull()
        this.$emit('touchend', event)
      })
      rendition.hooks.content.register((contents) => {
        contents.document.addEventListener('touchcancel', this.resetScrolledPull, { passive: true })
      })

      this.applyDisplaySettings()
    },
    /** @param {string} flow */
    setFlow(flow) {
      if (flow === this.currentFlow) return
      console.log(`[EpubReader] Changing flow from ${this.currentFlow} to ${flow}`)

      // Changing the manager needs a new rendition. Stay on the current location
      this.inittingDisplay = true
      const cfi = this.currentLocationCfi
      this.resetScrolledPull()
      this.rendition.destroy()
      document.getElementById('viewer').innerHTML = ''

      this.createRendition(flow)
      this.rendition.display(cfi).then(() => {
        this.inittingDisplay = false
      })
    },
    /** @param {TouchEvent} e - touchmove from inside the book */
    handleScrolledPull(e) {
      const container = this.rendition?.manager?.container
      if (!container || this.currentFlow !== 'scrolled' || this.pullTriggered) return

      // screenY, because the iframe moves with the content while pulling
      const y = e.touches[0].screenY
      const atEnd = container.scrollTop + container.clientHeight >= container.scrollHeight - 1
      const atStart = container.scrollTop <= 0
      let edge = null
      if (atEnd && atStart) edge = y < this.pullTouchStartY ? 'end' : 'start'
      else if (atEnd) edge = 'end'
      else if (atStart) edge = 'start'

      // A pull only counts from the moment the reader is already at an edge, so normal scrolling is unaffected
      if (edge !== this.pullEdge) {
        this.pullEdge = edge
        this.pullStartY = y
      }
      if (!edge) {
        this.clearPullHold()
        this.pullOffset = 0
        this.pullActive = false
        return
      }

      this.pullActive = true
      this.pullCanNavigate = this.hasSection(edge)

      const distance = Math.max(edge === 'end' ? this.pullStartY - y : y - this.pullStartY, 0)
      const offset = Math.min(distance * PULL_RESISTANCE, PULL_MAX_OFFSET)
      this.pullOffset = edge === 'end' ? -offset : offset

      if (distance < PULL_ARM_DISTANCE || !this.pullCanNavigate) {
        this.clearPullHold()
        return
      }

      if (!this.pullArmed) {
        this.pullArmed = true
        this.pullHoldTimeout = setTimeout(() => this.completePull(edge), PULL_HOLD_MS)
      }
    },
    /** @param {'start' | 'end'} edge */
    hasSection(edge) {
      const index = this.rendition?.currentLocation()?.start?.index
      if (index === undefined || !this.book) return false
      return edge === 'end' ? index < this.book.spine.length - 1 : index > 0
    },
    /** @param {'start' | 'end'} edge */
    completePull(edge) {
      this.pullHoldTimeout = null
      this.pullArmed = false
      if (!this.rendition) return

      this.pullTriggered = true
      const navigation = edge === 'end' ? this.rendition.next() : this.rendition.prev()
      // Spring back over the new chapter once it has rendered
      Promise.resolve(navigation).finally(() => this.resetPullVisuals())
    },
    clearPullHold() {
      clearTimeout(this.pullHoldTimeout)
      this.pullHoldTimeout = null
      this.pullArmed = false
    },
    resetPullVisuals() {
      this.clearPullHold()
      this.pullOffset = 0
      this.pullActive = false
      this.pullEdge = null
      this.pullStartY = null
      this.pullCanNavigate = false
    },
    resetScrolledPull() {
      this.resetPullVisuals()
      this.pullTriggered = false
    },
    applyDisplaySettings() {
      const fontScale = this.ereaderSettings.fontScale || 100
      this.rendition.themes.fontSize(`${fontScale}%`)
      this.rendition.themes.font(this.ereaderSettings.font)

      if (this.currentFlow === 'paginated') {
        this.rendition.spread(this.ereaderSettings.spread || 'auto')
      }
    },
    applyTheme() {
      if (!this.rendition) return
      this.rendition.getContents().forEach((c) => {
        c.addStylesheetRules(this.themeRules)
      })
    },
    async screenOrientationChange() {
      if (this.isRefreshingUI) return
      this.isRefreshingUI = true
      const windowWidth = window.innerWidth
      this.refreshUI()

      // Window width does not always change right away. Wait up to 250ms for a change.
      // iPhone 10 on iOS 16 took between 100 - 200ms to update when going from portrait to landscape
      //   but landscape to portrait was immediate
      for (let i = 0; i < 5; i++) {
        await new Promise((resolve) => setTimeout(resolve, 50))
        if (window.innerWidth !== windowWidth) {
          this.refreshUI()
          break
        }
      }

      this.isRefreshingUI = false
    },
    refreshUI() {
      if (!this.rendition?.resize) return

      // Showing or hiding the status bar changes innerHeight but not the layout. Resizing then rebuilds the views and jumps the scroll position, so only resize for width or player changes
      if (window.innerWidth === this.renditionWidth && this.readerHeightOffset === this.renditionHeightOffset) return

      this.renditionWidth = window.innerWidth
      this.renditionHeightOffset = this.readerHeightOffset
      this.rendition.resize(window.innerWidth, window.innerHeight - this.readerHeightOffset)
    }
  },
  mounted() {
    this.initEpub()

    if (screen.orientation) {
      // Not available on ios
      screen.orientation.addEventListener('change', this.screenOrientationChange)
    } else {
      document.addEventListener('orientationchange', this.screenOrientationChange)
    }
    window.addEventListener('resize', this.screenOrientationChange)
  },
  beforeDestroy() {
    this.clearPullHold()
    this.book?.destroy()

    if (screen.orientation) {
      // Not available on ios
      screen.orientation.removeEventListener('change', this.screenOrientationChange)
    } else {
      document.removeEventListener('orientationchange', this.screenOrientationChange)
    }
    window.removeEventListener('resize', this.screenOrientationChange)
  }
}
</script>

<style>
#epub-frame {
  height: calc(100% - 32px);
  max-height: calc(100% - 32px);
  overflow: hidden;
}
.reader-player-open #epub-frame {
  height: calc(100% - 132px);
  max-height: calc(100% - 132px);
  overflow: hidden;
}

.pull-progress {
  animation: pull-fill 500ms linear forwards;
}
@keyframes pull-fill {
  from {
    width: 0;
  }
  to {
    width: 100%;
  }
}
</style>
