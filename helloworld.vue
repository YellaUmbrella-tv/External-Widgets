<template>
  <!-- The template section is where you design your widget using HTML -->
  <div></div>
</template>

<script>
module.exports = {
  data: () => {
    return {
        // referenced in the updateConfig function as an example.
        settings: {
            mingap: 5
        },
        // Here you can specify variables to be used throughout the vue widget.    
        subslist: [],
        selectedtag: '',
        selectedtag2: '',
        newtag: '',
        translating: '',
        error: '',
    };
  },
  watch: {
    selectedtag: function () {},
  },
  methods: {
    action: function (sourcefile, destfile) {
        // Using the confirm dialog, you can check if the user confirms the action before executing
        // If the result of the dialog is Yes, the callback continues and the file is manipulated.
        this.launchConfirmDialog('Would you like to continue?', () => {
          // manipulate file from here.
          console.log(sourcefile, destfile);
          this.loaderDialogRef.close();
        })
    },

    // If your external widget has settings in it's config. The updateConfig function runs when you hit save.
    // You can use this to update local variables within the vue component.
    updateConfig: function () {
      console.log('make ad from subs config changed');
      if (this.appScope.widgetConfig && this.appScope.widgetConfig.data) {
        if (
          this.settings.mingap !== this.appScope.widgetConfig.data.mingap &&
          this.appScope.widgetConfig.data.mingap !== undefined
        ) {
          this.settings.mingap = +this.appScope.widgetConfig.data.mingap;
        }
      }
    },

    onClickAlign: function () {
      this.sourcefile = this.menuX(this.dialogContent, 'sourcefile', 'value');
      this.destfile = this.menuX(this.dialogContent, 'destfile', 'value');
      if (this.sourcefile && this.destfile) {
        this.action(this.sourcefile, this.destfile);
      }
    },

    // A custom function that runs when the dialog opens.
    onLaunchCustomDialog: function (item) {
      // This actually loads the dialog with the appropriate dialogContent.
      this.loaderDialogRef = this.appScope.dialogService.openDialog(
        'Custom',
        null,
        this.dialogContent,
        this.onDialogClosed.bind(this),
      );
    },

    // Helper dialog for confirmation.
    launchConfirmDialog(message, cb, cancelcb = null) {
        this.appScope.dialogService.openDialog(
        'Confirm',
        null,
        {
            message,
            closeButton: 'No',
            parent: this,
        },
        result => result ? cb() : cancelcb && cancelcb()
        );
    },

    // File browse dialog for the source file
    onClickSourcefile: function () {
      let srcfile = this.menuX(this.dialogContent, 'sourcefile', 'value');

      let srcpath = '';
      if (srcfile) {
        let srcsplt = srcfile.split('/');
        if (srcsplt.length > 1) {
          srcsplt.pop();
          srcpath = srcsplt.join('/');
        }
      }
      let srclibrary = 'files';
      let libraries = [{ name: 'files' }];

      // this is SaveAs, so set readonly to false explicitly
      let destDialogData = {
        isReadonly: true,
        index: 0,
        browseResults: {
          //tab: this.extraInfo[item].tab,
          fileLocation: srclibrary,
          file: srcfile,
          path: srcpath,
          exts: undefined,
          OK: false,
          type: '',
        },
        subMenuItems: [
          {
            name: 'Browse',
            id: 'browse',
            that: null,
            function: null, // not used here
            content: {
              type: 'browse',
            },
          },
        ],
        libraries: libraries,
        defaultLibrary: srclibrary,
        parent: this,
        filetype: [],
        selectedType: undefined,
        defaultType: undefined,
      };

      const exts = [];
      // BE SURE TO ONLY PUSH LOWER CASE
      destDialogData.defaultLibrary = 'files';

      // push filetypes you'd like to see
      exts.push('srtad');
      exts.push('esf');
      exts.push('pac');
      exts.push('fpc');
      exts.push('stl');
      exts.push('srt');
      exts.push('sif');
      exts.push('xml');
      exts.push('ttml');
      exts.push('dfxp');
      exts.push('scc');
      exts.push('vtt');
      destDialogData.browseResults.exts = exts;

      // function defined here... not in class.
      const onSaveAsDialogClosed = function () {
        // note: we can access dialogdata and dialogData.browseResults
        console.log('close browse of destination ', destDialogData);
        if (destDialogData.browseResults.OK) {
          let file = destDialogData.browseResults.file.split('.');
          this.menuX(
            this.dialogContent,
            'sourcefile',
            'value',
            destDialogData.browseResults.file,
          );
          this.menuX(
            this.dialogContent,
            'sourcefile',
            'library',
            destDialogData.browseResults.fileLocation,
          );
          this.menuX(
            this.dialogContent,
            'sourcefile',
            'filetype',
            destDialogData.browseResults.type,
          );
        }
      };

      console.log('open browse of src ', destDialogData);

      destDialogRef = this.appScope.dialogService.openDialog(
        'FileBrowse',
        null,
        destDialogData,
        onSaveAsDialogClosed.bind(this),
      ); // note function created above
    },

    onClickDestinationfile: function () {
      let srcfile = this.menuX(this.dialogContent, 'sourcefile', 'value');
      let srcpath = '';
      let srcsplt = srcfile.split('/');
      if (srcsplt.length > 1) {
        srcsplt.pop();
        srcpath = srcsplt.join('/');
      }
      let srclibrary = 'files';
      let libraries = [{ name: 'files' }];

      // setup initial folder and name from src file, if not already set in menu option

      // this includes path

      let destDialogRef = undefined;

      // this is SaveAs, so set readonly to false explicitly
      let destDialogData = {
        isReadonly: false,
        index: 0,
        browseResults: {
          //tab: this.extraInfo[item].tab,
          fileLocation: srclibrary,
          file: srcfile,
          path: srcpath,
          exts: undefined,
          OK: false,
          type: '',
        },
        subMenuItems: [
          {
            name: 'Browse',
            id: 'browse',
            that: null,
            function: null, // not used here
            content: {
              type: 'browse',
            },
          },
        ],
        libraries: libraries,
        defaultLibrary: srclibrary,
        parent: this,
        filetype: undefined,
        //path : this.path,
        selectedType: undefined,
        defaultType: undefined,
      };

      let filetypes = [];
      let defaulttype = '';
      let masterfiletypes = this.appScope.textEditorService.masterfiletypes;

      const exts = [];
      // BE SURE TO ONLY PUSH LOWER CASE
      destDialogData.defaultLibrary = 'files';
      filetypes.push(masterfiletypes.ad['srtad']);
      filetypes.push(masterfiletypes.ad['esef']);

      exts.push('srtad');
      exts.push('esf');

      filetypes.push(masterfiletypes.subtitle['pac']);
      filetypes.push(masterfiletypes.subtitle['fpc']);
      filetypes.push(masterfiletypes.subtitle['ebustl']);
      filetypes.push(masterfiletypes.subtitle['srt']);
      filetypes.push(masterfiletypes.subtitle['sif']);
      filetypes.push(masterfiletypes.subtitle['xmlttml']);
      filetypes.push(masterfiletypes.subtitle['ttml']);
      filetypes.push(masterfiletypes.subtitle['dfxp']);
      filetypes.push(masterfiletypes.subtitle['scc']);
      filetypes.push(masterfiletypes.subtitle['vtt']);

      defaulttype = masterfiletypes.subtitle['srt'].type;

      exts.push('pac');
      exts.push('fpc');
      exts.push('stl');
      exts.push('srt');
      exts.push('sif');
      exts.push('xml');
      exts.push('ttml');
      exts.push('dfxp');
      exts.push('scc');
      exts.push('vtt');

      destDialogData.filetype = filetypes;
      destDialogData.defaultType = defaulttype;
      destDialogData.selectedType = defaulttype;
      destDialogData.browseResults.exts = exts;

      // function defined here... not in class.
      const onSaveAsDialogClosed = function () {
        // note: we can access dialogdata and dialogData.browseResults
        console.log('close browse of destination ', destDialogData);
        if (destDialogData.browseResults.OK) {
          this.menuX(
            this.dialogContent,
            'destfile',
            'value',
            destDialogData.browseResults.file,
          );
          this.menuX(
            this.dialogContent,
            'destfile',
            'library',
            destDialogData.browseResults.fileLocation,
          );
          this.menuX(
            this.dialogContent,
            'destfile',
            'filetype',
            destDialogData.browseResults.type,
          );
        }
      };

      console.log('open browse of destination ', destDialogData);

      destDialogRef = this.appScope.dialogService.openDialog(
        'FileBrowse',
        null,
        destDialogData,
        onSaveAsDialogClosed.bind(this),
      ); // note function created above
    },

    onDialogClosed: function (res) {
      console.log('align dialog closed');
      this.fullscreen = false;
    },

    onDialogOpen: function () {
      console.log('align dialog open');
      console.log(this.loaderDialogRef);
    },

    onDialogOpened: function () {
      console.log('align dialog opened');
      console.log(this.loaderDialogRef);
    },

    onDialogClose: function () {
      console.log('align dialog close button');
    },

    onDialogChange: function () {
      console.log('align dialog change');
    },

    // functions to change menu items by id
    // set a key in a menu item, and return 1 if it was updated or added.
    // i.e. use the return to decide if menu needs re-render
    menuX: function (dialog, id, key, value) {
      var res = 0;
      if (!this.menuCacheX) {
        this.menuCacheX = {};
      }
      if (!this.menuCacheX[dialog.name]) {
        this.menuCacheX[dialog.name] = {};
        for (var i = 0; i < dialog.menuItems.length; i++) {
          this.menuCacheX[dialog.name][dialog.menuItems[i].id] = dialog.menuItems[i];
        }
      }

      if (this.menuCacheX[dialog.name][id]) {
        // if write
        if (value !== undefined) {
          if (this.menuCacheX[dialog.name][id][key] !== value) {
            this.menuCacheX[dialog.name][id][key] = value;
            res = 1;
          } else {
            res = 0;
          }
        } else {
          // read
          return this.menuCacheX[dialog.name][id][key];
        }
      }
      return res;
    },
  },

    // mounted occurs when the vue component is loaded.
  mounted() {
    this.localdata = {};
    this.localfunctions = {};
    var l = this.localdata;
    var f = this.localfunctions;
    // The appScope contains the services available to the external widget,
    // more information will be available about these services and the functions they provide
    var appScope = this.getAppScope();

    // solve issue of appScope.
    this.appScope = appScope;
    f.appScope = appScope;
    l.subtitlesService = this.appScope.subtitlesService;
    l.data = {};

    // adding services to localdata for referencing throughout the vue component.
    l.timecodeService = this.appScope.timecodeService;
    l.mediaControlService = this.appScope.mediaControlService;
    l.textEditorService = this.appScope.textEditorService;

    l.translatedest = this.appScope.elem.nativeElement.querySelector('#translatedest');

    l.timesubscription = l.mediaControlService.mediaObjectTime.subscribe(time => {
      const currentTime = time;
      if (l.globalTimeline !== currentTime) {
        l.globalTimeline = currentTime;
      }
    });

    this.UpdateSubscription$ = this.appScope.subtitlesService.updates$.subscribe(content => {
      // expect { changes: [{type:'endUpdate'}] } to trigger full load
      if (content.changes) {
        for (var i = 0; i < content.changes.length; i++) {
          var change = content.changes[i];
          switch (change.type) {
            case 'endUpdate':
              break;
          }
        }
      }
    });

    // defNavItems is the structure that adds items to the nav bar at the top.
    this.defNavItems = [
      {
        displayName: 'Tools',
        //style:{ backgroundColor: 'red', textDecoration: 'line-through' },
        id: 'helloworlddialog',
        title: 'Show Tools Menu',
        order: 20,
        children: [
          {
            displayName: 'Hello World',
            // style:{ color: 'red' },
            id: 'helloworlddialog',
            iconName: 'note',
            order: 20,
            title: 'Reads a file and writes it to a new file.',
            data: {
              onClick: this.onLaunchCustomDialog.bind(this),
            },
          },
        ],
      },
    ];

    // Adds the new navigation items to the top bar via the topMenuService.
    this.appScope.topMenuService.addNavItems(this.defNavItems);

    // Adds the Hello World option the the right click context menu of a column header.
    this.customDialogMenuItems = [
      {
        displayName: 'Hello World',
        id: 'helloworld',
        title: 'Manipulate this file',
        data: {
          onClick: this.onLaunchCustomDialog.bind(this)
        }
      },
    ];

    // Getting a unique id by using the id of this vue component.
    const id = this.appScope.config.id;
    this.appScope.contextMenuService.addMenuItems(this.customDialogMenuItems, 'columnheader', id);
    
    this.dialogContent = {
      name: 'helloworlddialog',
      headerTitle: 'Hello World!',
      title: 'This is an example external widget.',
      width: '60%',
      closeButton: true,
      //height: '60%',
      // maxWidth: '80%',
      onOpen: this.onDialogOpen.bind(this), // You can bind custom functions to these resulting actions.
      onClose: this.onDialogClose.bind(this),
      onChange: this.onDialogChange.bind(this),
      updateFunction: null,
      // menuItems is where you construct the options within a dialog
      menuItems: [
        {
          type: 'file',
          id: 'sourcefile',
          placeholder: 'Select first file.',
          label: 'Input File',
          readonly: false,
          value: '',
          onClick: this.onClickSourcefile.bind(this),
          title: 'Lists and allows selection from available files',
        },
        {
          type: 'savefile',
          id: 'destfile',
          placeholder: '',
          label: 'Output File',
          readonly: false,
          value: '',
          onClick: this.onClickDestinationfile.bind(this),
          title: 'Path where new file will be created',
        },
        {
          type: 'button',
          id: 'render',
          label: 'Action',
          onClick: this.onClickAlign.bind(this),
          title: 'Write the first file to the new file.',
        },
      ],
    };
  },
  // Before the vue component is unloaded you need to unsubscribe to any observables you've subscribed to.
  // Otherwise the subscription will continue to recieve data when the component is unloaded.
  beforeDestroy() {
    var l = this.localdata;
    l.timesubscription.unsubscribe();
    this.UpdateSubscription$.unsubscribe();
    // You can also remove the context and top menu items here too.
    this.appScope.topMenuService.removeNavItems(this.defNavItems);
    this.appScope.contextMenuService.removeMenuItems(this.customDialogMenuItems, 'columnheader');
  },
  destroyed() {},
  CONFIG: {
    name: 'Example External Widget',
    selector: 'stellar-external-widget-container:helloworld',
    headless: true, // has no widget
    // Here you can set the default height and width if there is a widget associated.
    defaultConfig: {
      config: {
        cols: 60,
        rows: 30,
        y: 0,
        x: 0,
        hasContent: true,
      },
      data: {},
      widgetConfigs: [],
    },
  },
};
</script>

<style scoped>

</style>
