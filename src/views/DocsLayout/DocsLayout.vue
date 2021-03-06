<template lang="pug">
.docsLayout(:class="{ 'docsLayout--header-hidden': !showHeader, 'docsLayout--sidebar-hidden': !sidebarLinks.length, 'docsLayout--device-menu-visible': showDeviceMenu }")

  //- Header Area
  AppHeader.docsLayout__header#header(
    @toggleMenu="toggleMenu"
    :showMenu="showDeviceMenu"
    :mode="mode"
    :title="title")
    template(slot="headerSearch")
      //- Dark/Light Mode
      AppThemeMode(
        v-if="showToggleMode"
        :mode="mode"
        @toggleMode="setMode")

      //- Buscador
      Search.appHeader__search
    template(slot="headerMenu")
      AppMenu(
        ref="headerMenu"
        :links="navbarLinks"
        v-observe-visibility="{callback: closeHeaderMenus, intersection: { threshold: 0.9 }}")

  //- Sidebar Area
  AppSidebar.docsLayout__sidebar(:links="sidebarLinks")


  //- Scroll Tracking
  ScrollTracking(:blocks="toc" v-if="content && toc && toc.length > 1")


  //- Main Content
  .docsLayout__main#main(ref="main")
    .docsLayout__wrapper(v-if="content")

      //- Article Area
      AppArticle.docsLayout__article(:data="content")


      //- Footer Area
      AppPagination.docsLayout__pagination(
        v-if="article"
        :prevArticle="article.prev"
        :nextArticle="article.next")

    AppNotFound(:title="title" v-if="notFound")


  //- Botón para mostrar/ocultar el menú lateral
  Animation(name="fadeDown")
    .docsLayout__device-menu(v-if="showDeviceMenu")
      AppMenu(:links="navbarLinks" vertical)
      AppSidebar(:links="sidebarLinks" v-if="sidebarLinks.length")
      AppSocialLinks
</template>

<script>
import AppHeader from '@/components/AppHeader';
import AppSidebar from '@/components/AppSidebar';
import AppMenu from '@/components/AppMenu';
import AppSocialLinks from '@/components/AppSocialLinks';
import AppArticle from '@/components/AppArticle';
import AppPagination from '@/components/AppPagination';
import AppNotFound from '@/components/AppNotFound';
import AppThemeMode from '@/components/AppThemeMode';
import Search from '@/components/Search.component.vue';
import ScrollTracking from '@/components/ScrollTracking';
import { ObserveVisibility } from 'vue-observe-visibility';
import FileService from '@/services/file.service';
import ParamsUtil from '@/utils/params.utils';
import ArrayUtils from '@/utils/array.utils';

export default {
  directives: {
    ObserveVisibility,
  },
  components: {
    AppHeader,
    AppSidebar,
    AppMenu,
    AppSocialLinks,
    AppArticle,
    AppPagination,
    AppNotFound,
    AppThemeMode,
    Search,
    ScrollTracking,
  },
  data() {
    return {
      logo: '',
      title: '',
      social: '',
      defaultPath: '',
      notFound: false,
      navbarPath: '',
      sidebarPath: '',
      section: null,
      article: null,
      showHeader: true,
      showDeviceMenu: false,
      lastScrollPosition: 0,
      navbarLinks: [],
      sidebarLinks: [],
      content: '',
      toc: [],
      mode: 'light',
      showToggleMode: false,
    };
  },
  methods: {
    toggleMenu() {
      this.showDeviceMenu = !this.showDeviceMenu;
    },

    closeHeaderMenus() {
      this.$refs.headerMenu.closeAllDropdown();
    },

    onScroll() {
      const currentScrollPosition = window.scrollY;
      if (currentScrollPosition < 0) {
        return;
      }

      // Stop executing this function if the difference between
      // current scroll position and last scroll position is less than some offset
      if (Math.abs(currentScrollPosition - this.lastScrollPosition) < 85) {
        return;
      }

      this.showHeader = currentScrollPosition < this.lastScrollPosition;
      this.lastScrollPosition = currentScrollPosition;
    },

    /**
     * Obtiene los datos para mostrar la documentación
     *
     * @params {String} routeTo - Ruta de destino
     * @params {String} routeFrom - Ruta de origen
     */
    async getData(routeTo, routeFrom, hash) {
      /**
       * Si la ruta viene con un "/" al final,
       * da problemas al comparar la ruta
       */
      if (routeTo !== '/' && routeTo.slice(-1) === '/') {
        routeTo = routeTo.slice(0, -1);
      }

      /**
       * Si no se especifica ninguna ruta y tengo un artículo por defecto
       * redirijo a dicho artículo
       */
      if (routeTo === '/' && this.defaultPath) {
        this.$router.push(this.defaultPath, () => {});
        return;
      }

      /**
       * Cuando tenemos secciones y no se especifica ninguna ruta, redirecciono a la primera sección
       * El listado 'this.navbarLinks' puede contener una estructura anidada
       * para asegurarme que obtengo el primer link, lo busco en una estructura plana
       */
      if (routeTo === '/' && ArrayUtils.checkArray(this.navbarLinks)) {
        const flatNavbarLinks = ArrayUtils.flattenList(this.navbarLinks);
        const firstSection = flatNavbarLinks[0].link;
        this.$router.push(firstSection, () => {});
        return;
      }

      /**
       * Si la sección ha cambiado, obtenemos la nueva sección y el sidebar
       */
      const fromSection = ParamsUtil.getSection(routeFrom);
      const toSection = ParamsUtil.getSection(routeTo);

      if (fromSection !== toSection) {
        /**
         * Si la sección ha cambiado, inicializo los valores
         */
        this.section = null;
        this.sidebarLinks = [];

        /**
         * Si tenemos secciones, busco la sección actual,
         * si no existe, es que esta página no tiene navbar
         */
        if (ArrayUtils.checkArray(this.navbarLinks)) {
          const section = await ArrayUtils.searchItemByLinks(this.navbarLinks, toSection);
          this.section = section ? section.current : null;
        }

        /**
         * Compruebo si la sección actual es un archivo o un directorio
         * si es un directorio, intento obtener el listado de artículos
         */
        const sectionIsFolder = this.section && this.section.file.slice(-3) !== '.md';

        if (sectionIsFolder) {
          const sectionLink = this.section.link;
          let sidebarPath = `${this.section.file}/${this.sidebarPath}`;

          // Si el usuario modifica el nombre del archivo por defecto de la sidebar
          // y quiere mostrar didorDocs o didorStyles, tengo que modificar su path
          // para que coincida con el de ellos que usan el de por defecto
          if ((this.section.file === '/didorDocs' && this.showDidorDocs) || (this.section.file === '/didorStyles' && this.showDidorStyles)) {
            sidebarPath = `${this.section.file}/_sidebar.md`;
          }

          this.sidebarLinks = await FileService.getLinks(sidebarPath, this.buildPath, sectionLink);
        }
      }

      /**
       * Si tenemos un listado de artículos, busco el artículo actual
       */
      this.article = null;
      this.content = '';
      this.deviceShow = false;

      if (ArrayUtils.checkArray(this.sidebarLinks)) {
        /**
         * Cuando no se especifica ninguna ruta o la ruta es una sección,
         * redirecciono al primer artículo de la sección
         * El listado 'this.sidebarLinks' puede contener una estructura anidada
         * para asegurarme que obtengo el primer link, lo busco en una estructura plana
         */
        if (routeTo === '/' || (this.section && this.section.link === routeTo)) {
          const flatSidebarLinks = ArrayUtils.flattenList(this.sidebarLinks);
          const firstArticle = flatSidebarLinks[0].link;
          this.$router.push(firstArticle, () => {});
          return;
        }

        this.article = await ArrayUtils.searchItemByLinks(this.sidebarLinks, routeTo);
      }

      /**
       * Obtenemos el contenido del árticulo y los datos
       */
      let file = '';

      if (this.article) file = this.article.current.file;
      else if (this.section) file = this.section.file;
      else if (routeTo !== '/') file = `${routeTo}.md`;

      const content = file ? await FileService.getArticle(file, this.buildPath) : null;

      if (content) {
        this.content = content.render ? content.render : '';
        this.deviceShow = content.data && content.data.device ? content.data.device : false;
        this.toc = content.data && content.data.toc ? content.data.toc : [];
      } else {
        // Artículo no encontrado
        this.notFound = true;
      }

      /**
       * Si existe un hash hago scroll hasta su posición
       * tengo que esperar un poco para que se renderizo el contenido
       * y poder hacer scroll hasta el título
       */
      if (hash) this.scrollToHash(hash);

      /**
       * Actualizo el título de la página
       */
      const title = this.article ? this.article.current.title : this.section ? this.section.title : '';
      document.title = `${this.title} - ${title}`;
    },

    /**
     * Obtiene el modo de la aplicación
     */
    getMode() {
      //
      // Obtengo el modo definido en los ajustes y almacenado en local
      //
      const settingMode = window.$didor.customize.mode;
      const localMode = localStorage.getItem('didor-theme');
      const systemMode = window.matchMedia && window.matchMedia('(prefers-color-scheme: dark)').matches ? 'dark' : 'light';

      //
      // Asigno el modo dando preferencia al local,
      // después al de los ajustes de la documentación
      // luego al del sistema
      // y si no obtengo ninguno, utilizo el 'light' por defecto
      //
      const mode = localMode || settingMode || systemMode || 'light';

      this.setMode(mode);
    },

    /**
     * Actualiza el modo de la aplicación
     */
    setMode(mode) {
      this.mode = mode;
      document.getElementsByTagName('html')[0].setAttribute('data-theme', mode);
      localStorage.setItem('didor-theme', mode);
    },

    /**
     * Hace scroll hasta el hash seleccionado.
     */
    scrollToHash(hash) {
      setTimeout(() => {
        hash = hash.replace('#', '');

        const headerHeight = document.getElementById('header').offsetHeight;
        const anchorTop = document.getElementById(hash).offsetTop;

        window.scroll(0, anchorTop - headerHeight);
      }, 100);
    },
  },

  async created() {
    /**
     * Obtengo la configuración del proyecto
     */
    const config = window.$didor;

    this.title = config.customize.title;
    this.description = config.customize.description;
    this.logo = config.customize.logo;
    this.social = config.customize.social;
    this.navbarPath = config.content.navbar;
    this.sidebarPath = config.content.sidebar;
    this.defaultPath = config.content.defaultPath
      .split('.')
      .slice(0, -1)
      .join('.');
    this.buildPath = config.build.path;
    this.showDidorDocs = config.content.didorDocs;
    this.showDidorStyles = config.content.didorStyles;
    this.showToggleMode = window.$didor.customize.toggleMode;

    /**
     * Obtengo y aplico el modo de visualización de la app: 'light' o 'dark'
     */
    this.getMode();

    /**
     * Actualizo el título de la página con el nombre del proyecto
     */
    document.title = this.title;
    document.querySelector('meta[name="description"]').setAttribute('content', this.description);

    /**
     * Intento obtener los links de la secciones
     */
    this.navbarLinks = await FileService.getLinks(this.navbarPath, this.buildPath);

    /**
     * Obtengo los datos de la ruta
     */
    this.getData(this.$route.path, null, this.$route.hash);
  },

  mounted() {
    window.addEventListener('scroll', this.onScroll, false);
  },

  beforeDestroy() {
    window.removeEventListener('scroll', this.onScroll);
  },

  async beforeRouteUpdate(routeTo, routeFrom, next) {
    /**
     * Siempre que haya un cambio de ruta, cierro el menú en los dispositivos móviles
     */
    this.showDeviceMenu = false;
    this.notFound = false;

    /**
     * Compruebo si la ruta ha combiado para evitar que recargue la página
     * cuando hago click sobre un enlace dentro de la misma vista (ancla).
     *
     */
    if (routeTo.path === routeFrom.path || (routeTo.path === '/' && routeFrom.path === this.defaultPath)) {
      if (routeTo.hash) this.scrollToHash(routeTo.hash);

      return;
    }

    /**
     * Al ser siempre la misma ruta, el método scroll behavior de VueRouter no sirve,
     * así que tengo que hacerlo manualmente.
     * Desactivo temporalmente la propiedad css scrollBeahavior,
     * para que vaya de un salto y no con una animación al cambiar de página.
     */
    const main = document.body;
    main.style.scrollBehavior = 'initial';
    setTimeout(() => {
      main.scroll(0, 0);
      main.style.scrollBehavior = 'smooth';
    }, 100);

    /**
     * Por defecto, si la ruta de destino apunta a una sección y no a un artículo
     * redirijo al primer artículo de dicha sección.
     * El problema es que si estoy en el primer artículo de la sección,
     * y pulso en algún enlace que apunte a la sección,
     * al intentar ir a la misma ruta en la que me encuentro se genera un error.
     * Para evitarlo, compruebo cuando se dá el caso y evito que la redirección.
     */

    /**
     * Cuando tenemos secciones:
     * Si la ruta de destino apunta a la sección actual,
     * compruebo si la ruta de ORIGEN coincide con el primer artículo de la sección,
     * en cuyo caso no debo realizar la redirección.
     */
    const sectionLink = this.section && this.section.link ? this.section.link : '/';

    if (this.sidebarLinks && routeTo.path === sectionLink) {
      const flatSidebarLinks = ArrayUtils.flattenList(this.sidebarLinks);
      const firstSection = flatSidebarLinks[0].link;
      if (routeFrom.path === firstSection) return;
    }

    /**
     * Cuando no tenemos secciones y tenemos un listado de artículos:
     * Si la ruta de destino apunta a la raíz, se cargará por defecto el primer artículo,
     * compruebo si la ruta de ORIGEN coincide con el primer artículo de la sección,
     * en cuyo caso no debo realizar la redirección.
     */
    if (!this.sidebarLinks && this.navbarLinks && routeTo.path === '/') {
      const flatNavbarLinks = ArrayUtils.flattenList(this.navbarLinks);
      const firstArticle = flatNavbarLinks[0].link;
      if (routeFrom.path === firstArticle) return;
    }

    /**
     * Obtengo los datos de la nueva ruta
     */
    await this.getData(routeTo.path, routeFrom.path, this.$route.hash);

    next();
  },
};
</script>

<style src="./DocsLayout.scss" lang="scss" scoped></style>
