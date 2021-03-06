<template>
  <div class="ebook-reader">
    <div id="read"></div>
    <div
      class="ebook-reader-mask"
      @click="onMaskClick"
      @touchmove="move"
      @touchend="moveEnd"
      @mousedown.left="onMouseEnter"
      @mousemove.left="onMouseMove"
      @mouseup.left="onMouseEnd"
    ></div>
    <div class="empty" v-if="!bookAvailable">
      <ebook-loading></ebook-loading>
    </div>
  </div>
</template>

<script>
import Epub from "epubjs";
import EbookLoading from "./EbookLoading";
import { ebookMixin } from "@/mixins/ebook";
import { flatten } from "../../utils/book";
import {
  getFontFamily,
  saveFontFamily,
  getFontSize,
  saveFontSize,
  saveTheme,
  getTheme,
  getLocation,
  setCover
} from "../../utils/localStorage";
import { getLocalForage } from "../../utils/localForage";
//传给全局变量
global.ePub = Epub;
export default {
  name: "EbookReader",
  mixins: [ebookMixin],
  components: {
    EbookLoading
  },
  props: {},
  data() {
    return {
      book: null, // epub实例
      rendition: null, // 渲染epub
      mouseState: null,
      isOnOpf: false, // 是否从opf解析书籍
      firstOffsetY: null, //下来操作，touch第一次Y轴坐标
      mouseStartTime:0
    };
  },
  watch: {},
  computed: {},

  /**
   * epub.js的主要对象
   * book: 书籍的解析
   * rendition: 生成阅读器
   * locations: 书籍的定位
   * navigation: 书籍的封面作责出版社等信息
   * cfi: 对书籍进行文字级别的定位，如查找书籍中某一段文字
   * theme: 阅读器主题，字体等样式
   * spine: 管理section
   * section: 用于章节切换
   * annotations: 管理标签，如文字高亮
   */

  methods: {
    //初始化图书
    initEpub(url) {
      // let url = process.env.VUE_APP_EPUB_URL + this.fileName + ".epub";
      this.book = new Epub(url);
      this.setCurrentBook(this.book);
      this.parseBook();
      this.initRendition();
      // this.initGuest();
      //book解析完成后获取locations对象
      this.book.ready
        .then(() => {
          return this.book.locations.generate(
            750 * (window.innerWidth / 375) * (getFontSize(this.fileName) / 16)
          );
        })
        .then(locations => {
          // //获取分页
          // this.navigation.forEach(nav => {
          //   nav.pagelist = [];
          // });
          // locations.forEach(item => {
          //   const loc = item.match(/\[(.*)\]!/)[1];
          //   this.navigation.forEach(nav => {
          //     if (nav.href) {
          //       const href = nav.href.match(/^(.*)\.html$/);
          //       if (href) {
          //         if (href[1] === loc) {
          //           nav.pagelist.push(item);
          //         }
          //       }
          //     }
          //   });
          //   let currentPage = 1;
          //   this.navigation.forEach((nav, index) => {
          //     if (index === 0) {
          //       nav.page = 1;
          //     } else {
          //       nav.page = currentPage;
          //     }
          //     currentPage += nav.pagelist.length + 1;
          //   });
          // });
          // this.setPageList(location);

          //加载完成
          this.setBookAvailable(true);
          //更新阅读进度
          this.refreshLocation();
        });
    },

    //生成阅读器，并渲染书籍
    initRendition() {
      this.rendition = this.book.renderTo("read", {
        width: window.innerWidth,
        height: window.innerHeight,
        method: "default"
        // flow:'scrolled' //滚动模式,兼容不好
      });
      //判断是否从书籍详情页中某个章节阅读
      if (this.$route.query.navigation) {
        this.display(this.$route.query.navigation, () => {
          this.initTheme();
          this.initFontSize();
          this.initFontFamily();
          // this.refreshLocation();
          this.parseBook();
        });
      } else {
        //获取阅读历史，并渲染上一次阅读位置
        const location = getLocation(this.fileName);
        this.display(location, () => {
          this.initTheme();
          this.initFontSize();
          this.initFontFamily();
          // this.refreshLocation();
          this.parseBook();
        });
      }
      //载入字体文件
      this.rendition.hooks.content.register(contents => {
        let fontURL = process.env.VUE_APP_RES_URL + "/fonts/fonts.css";
        contents.addStylesheet(fontURL).then(() => {});
      });
    },

    //初始化主题
    initTheme() {
      this.registerTheme();
      let defaultTheme = getTheme(this.fileName);
      if (!defaultTheme) {
        defaultTheme = this.themeList[0].name;
        saveTheme(this.fileName, defaultTheme);
      } else {
        this.setDefaultTheme(defaultTheme);
      }
      this.rendition.themes.select(this.defaultTheme);
      this.setGlobalTheme(this.defaultTheme);
    },

    //初始化字体大小
    initFontSize() {
      let fontSize = getFontSize(this.fileName);
      if (!fontSize) {
        fontSize = this.defaultFontSize;
        saveFontSize(this.fileName, fontSize);
      } else {
        this.setDefaultFontSize(fontSize);
      }
      this.rendition.themes.fontSize(this.defaultFontSize);
    },

    //初始化字体样式
    initFontFamily() {
      let font = getFontFamily(this.fileName);
      if (!font) {
        font = this.defaultFontFamily;
        saveFontFamily(this.fileName, font);
      } else {
        this.setDefaultFontFamily(font);
      }
      this.rendition.themes.font(font);
    },

    //解析book,获取封面图片,作者等信息
    parseBook() {
      //获取封面
      if (this.isOnOpf) {
        this.book.coverUrl().then(url => {
          this.setCover(url);
        });
      } else {
        this.book.loaded.cover.then(cover => {
          this.book.archive.createUrl(cover).then(url => {
            this.setCover(url);
          });
        });
      }
      // 获取书籍信息
      this.book.loaded.metadata.then(metadata => {
        this.setMetadata(metadata);
      });
      // 获取书籍章节目录
      this.book.loaded.navigation.then(nav => {
        console.log("nav:", nav);
        //把目录转化为一维数组
        const navItem = flatten(nav.toc);
        console.log("navItem", navItem);
        //添加level属性,指定每项目录标是哪一级的目录
        function find(item, level = 0) {
          const parent = navItem.filter(parentItem => {
            parentItem.id == item.parent;
          })[0];
          if (!item.parent) {
            return level;
          } else {
            if (parent) {
              return find(parent, ++level);
            }
            return level;
          }
        }
        // 获取章节名称
        navItem.forEach((item, index) => {
          item.level = find(item);
          item.total = 0;
          item.pagelist = [];
          if (item.href.match(/^(.*)\.html$/)) {
            item.idhref = item.href.match(/^(.*)\.html$/)[1];
          } else if (item.href.match(/^(.*)\.xhtml$/)) {
            item.idhref = item.href.match(/^(.*)\.xhtml$/)[1];
          }
        });
        this.setNavigation(navItem);
      });
    },

    //上一页
    prevPage() {
      if (this.bookAvailable) {
        this.rendition.prev().then(() => {
          this.refreshLocation();
        });
        this.hideMenuVisible();
      }
    },

    //下一页
    nextPage() {
      if (this.bookAvailable) {
        this.rendition.next().then(() => {
          this.refreshLocation();
        });
        this.hideMenuVisible();
      }
    },

    //mask上的手势操作,翻页..
    onMaskClick(e) {
      if (
        this.mouseState &&
        (this.mouseState == 2 || this.mouseState == 3) &&
        !this.bookAvailable
      ) {
        return;
      }
      //获取鼠标点击的屏幕x轴坐标，根据坐标在屏幕位置执行事件
      const offsetX = e.offsetX;
      const width = window.innerWidth;
      if (offsetX > 0 && offsetX < width * 0.3) {
        this.prevPage();
      } else if (offsetX > 0 && offsetX > width * 0.7) {
        this.nextPage();
      } else {
        this.toggleMenuVisible();
      }
    },

    //下拉book
    move(e) {
      if (!this.bookAvailable) {
        return;
      }
      let offsetY = 0;
      if (this.firstOffsetY) {
        //获取下拉的距离
        offsetY = e.changedTouches[0].clientY - this.firstOffsetY;
        this.setOffsetY(offsetY);
      } else {
        this.firstOffsetY = e.changedTouches[0].clientY;
      }
      e.preventDefault();
      e.stopPropagation();
    },

    //下拉松手
    moveEnd(e) {
      if (!this.bookAvailable) {
        return;
      }
      this.setOffsetY(0);
      this.firstOffsetY = null;
    },

    // pc端书签功能
    //1-鼠标按下
    //2-鼠标按下后移动
    //3-鼠标从移动状态松手
    //4-鼠标还原

    onMouseEnter(e) {
      if (!this.bookAvailable) {
        return;
      }
      this.mouseState = 1;
      // 鼠标按下时的时间戳
      this.mouseStartTime = e.timeStamp;
      e.preventDefault();
      e.stopPropagation();
    },

    onMouseMove(e) {
      if (!this.bookAvailable) {
        return;
      }
      if (this.mouseState == 1) {
        this.mouseState = 2;
      } else if (this.mouseState == 2) {
        let offsetY = 0;
        if (this.firstOffsetY) {
          // 获取下拉的长度
          offsetY = e.clientY - this.firstOffsetY;
          this.setOffsetY(offsetY);
        } else {
          this.firstOffsetY = e.clientY;
        }
      }
    },

    onMouseEnd(e) {
      if (!this.bookAvailable) {
        return;
      }
      if (this.mouseState == 2) {
        this.setOffsetY(0);
        this.firstOffsetY = null;
        this.mouseState = 3;
      } else {
        this.mouseState = 4; //只点击不移动
      }
      //修正误触，鼠标按下时间小于400毫秒
      const time = e.timeStamp - this.mouseStartTime;
      if (time < 400) {
        this.mouseState = 4;
      }
      e.preventDefault();
      e.stopPropagation();
    }

    // //初始化book手势功能
    // initGuest() {
    //  this.rendition.on("touchstart", event => {
    //     this.touchStartX = event.changedTouches[0].clientX;
    //     this.touchStartTime = event.timeStamp;
    //   });
    //  this.rendition.on("touchend", event => {
    //     const offsetX = event.changedTouches[0].clientX - this.touchStartX;
    //     const time = event.timeStamp - this.touchStartTime;
    //     if (time < 500 && offsetX > 40) {
    //       this.prevPage();
    //     } else if (time < 500 && offsetX < -40) {
    //       this.nextPage();
    //     } else {
    //       this.toggleMenuVisible();
    //     }
    //     event.preventDefault();
    //     event.stopPropagation();
    //   });
    // }
  },
  created() {},
  mounted() {
    const books = this.$route.params.fileName.split("|");
    const fileName = books[1];
    //在indexDB中查看是否有缓存
    getLocalForage(fileName, (err, blob) => {
      if (!err && blob) {
        this.setFileName(books.join("/")).then(() => {
          this.isOnOpf = false;
          this.initEpub(blob);
        });
      } else {
        // 选择章节阅读
        if (this.$route.query.opf) {
          this.isOnOpf = true;
          this.initEpub(this.$route.query.opf);
        } else {
          this.setFileName(books.join("/")).then(() => {
            const url =
              process.env.VUE_APP_EPUB_URL + "/" + this.fileName + ".epub";
            this.isOnOpf = false;
            this.initEpub(url);
          });
        }
      }
    });
  }
};
</script>
<style lang="scss" scoped>
.ebook-reader {
  width: 100%;
  height: 100%;
  overflow: hidden;
  .ebook-reader-mask {
    position: absolute;
    left: 0;
    top: 0;
    height: 100%;
    width: 100%;
    z-index: 100;
  }
  .empty {
    position: absolute;
    left: 0;
    top: 0;
    z-index: 120;
    width: 100%;
    height: 100%;
    background-color: rgb(82, 81, 81);
    @include center;
    // font-size: 16px;
    // color: #333;
  }
}
</style>