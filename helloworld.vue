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
    };
  },
  watch: {
    selectedtag: function () {},
  },
  methods: {
    action: function (selectedtag, destfile) {
        // Using the confirm dialog, you can check if the user confirms the action before executing
        // If the result of the dialog is Yes, the callback continues and the file is manipulated.
        this.launchConfirmDialog('Would you like to continue?', () => {
          // manipulate file from here.
          let tl = this.appScope.subtitlesService.timelines[selectedtag];

          let content = tl.content;

          let ids = [];
          let index = [];
          for (let i = 0; i < content.length; i++) {
            // make a note of the ids and timings for later.
            ids.push(content[i].i);
            index.push({i: content[i].i, b: content[i].b, e: content[i].e});
          }

          // using the getContentText function you can get the data for each id passed in a tag.
          res = this.appScope.subtitlesService.getContentText(selectedtag, ids);

          // subscribing to the result allows you to recieve and manipulate the data.
          res.subscribe({
            next: content => {
              let data = [];
              // loop through the results and parse the text from each title.
              for (let i = 0; i < content.results.length; i++) {
                let d = content.results[i].content;
                if (typeof d === 'string') {
                  d = JSON.parse(d);
                }
                // parse the DOM structure into strings.
                let text = this.gettext(d.body);
                // store them in an array to be used when creating a new file.
                data.push(text);
              }

              // using the text and indexes create a string that represents the srt content.
              let srttext = '';
              for (let i = 0; i < data.length; i++) {
                srttext += '' + (i + 1) + '\r\n';
                srttext +=
                  this.toSrtTime(index[i].b) + ' --> ' + this.toSrtTime(index[i].e) + '\r\n';
                srttext += data[i] + '\r\n';
                srttext += '\r\n';
              } // end of loop

              // save a new file using the text editor service.
              this.appScope.textEditorService
                .writeFolderFile(`files/${destfile}`, srttext)
                .subscribe(content => {
                  // once saved, add to the current project if you want.
                  this.addNewToCurrentProject(
                    this.selectedtag, // original tag
                    this.selectedtag + '_copy', // new tag
                    destfile, // file path
                    'srt', // file type
                    () => {}, // callback
                  );
                });
            },
            error: err => console.error('hello world error: ' + err),
            complete: () => console.log('hello world complete'),
          });
          // close the dialog
          this.loaderDialogRef.close();
        })
    },

    // ========= Helper functions ============== //

    // seperate text from subtitleservice getTextContent result.
    gettext: function (stl) {
      var t = '';
      var keys = Object.keys(stl);
      for (var i = 0; i < keys.length; i++) {
        if (keys[i] === 'text') {
          if (t) {
            t = t + ' ';
          }
          t = t + stl.text.trim();
        }
      }

      if (stl.contents) {
        for (var i = 0; i < stl.contents.length; i++) {
          var t1 = this.gettext(stl.contents[i]);
          if (t1) {
            if (t) {
              t = t + ' ';
            }
            t = t + t1.trim();
          }
        }
      }
      return t;
    },

    // convert seconds to srt timings.
    toSrtTime: function (s) {
      let out = this.to2Digit(s / 3600);
      out += ':';
      out += this.to2Digit((s / 60) % 60);
      out += ':';
      out += this.to2Digit(s % 60);
      out += ',';
      out += this.to3Digit((s * 1000) % 1000);
      return out;
    },
    to2Digit: function (val) {
      val = '0' + (val >> 0);
      return val.slice(-2);
    },
    to3Digit: function (val) {
      val = '00' + (val >> 0);
      return val.slice(-3);
    },

    // Add a new file to the current project.
    addNewToCurrentProject: function (orgtag, newtag, fname, type, cb) {
      // as long as project widget is here then
      // this.subtitlesService.project contains current project

      // Create the new tag.
      if (this.appScope.subtitlesService.project.timelines[orgtag]) {
        this.appScope.subtitlesService.project.timelines[newtag] = {
          file: fname,
          offset: this.appScope.subtitlesService.project.timelines[orgtag].offset,
          type: type,
          offset2: this.appScope.subtitlesService.project.timelines[orgtag].offset2,
          maxaccess: 'all',
          access: 'all',
        };

        // Write the new tag to the project and update the page with a refresh.
        this.appScope.textEditorService
          .writeProject(this.appScope.subtitlesService.project)
          .subscribe(content => {
            this.appScope.textEditorService
              .saveProjectFile(this.appScope.subtitlesService.project.name)
              .subscribe(content => {
                this.appScope.subtitlesService.project = content;
                this.appScope.textEditorService.texteditorContentUpdate.next('refresh');
                if (cb) {
                  cb();
                }
              });
          });
      }
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

    // The function bound to the action button being clicked.
    onClickAction: function () {
      this.selectedtag = this.menuX(this.dialogContent, 'selectfile', 'value');
      this.destfile = this.menuX(this.dialogContent, 'destfile', 'value');
      if (this.selectedtag && this.destfile) {
        this.action(this.selectedtag, this.destfile);
      }
    },

    // get list of available timelines.
    loadTagList: function () {
      var tltags = Object.keys(this.appScope.subtitlesService.timelines);
      this.subslist = [];
      for (var i = 0; i < tltags.length; i++) {
        var tl = this.appScope.subtitlesService.timelines[tltags[i]];
        if (tl.metadata && tl.metadata.text) {
          this.subslist.push(tltags[i]);
        }
      }
      if (this.subslist.length) {
        this.selectedtag = this.subslist[0];
      } else {
        this.selectedtag = '';
      }
    },

    // A custom function that runs when the dialog opens.
    onLaunchCustomDialog: function (item) {
      this.loadTagList();
      if (item.parent && item.parent.data) {
        this.selectedtag = item.parent.data.timelineid;
      }
      
      this.menuX(this.dialogContent, 'selectfile', 'options', this.subslist);
      this.menuX(this.dialogContent, 'selectfile', 'value', this.selectedtag);

      if (
        (this.subslist.length > 0 && !this.selectedtag) ||
        (this.subslist.length > 0 && this.subslist.indexOf(this.selectedtag) < 0)
      ) {
        this.selectedtag = this.subslist[0];
        this.menuX(this.dialogContent, 'selectfile', 'value', this.subslist[0]);
      }

      this.onDialogChange();
      
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

      filetypes.push(masterfiletypes.subtitle['srt']);
      exts.push('srt');

      defaulttype = masterfiletypes.subtitle['srt'].type;
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
    },

    onDialogOpened: function () {
      console.log('align dialog opened');
      
    },

    onDialogClose: function () {
      console.log('align dialog close button');
    },

    onDialogChange: function () {
      let tag = this.menuX(this.dialogContent, 'selectfile', 'value');
      let file = '';
      if (tag && this.appScope.subtitlesService.project.timelines[tag]) {
        file = this.appScope.subtitlesService.project.timelines[tag].file;
      }
      this.selectedtag = tag;

      let srcfile = this.menuX(this.dialogContent, 'sourcefile', 'value');
      if (file !== srcfile) {
        this.menuX(this.dialogContent, 'sourcefile', 'value', file);
      }
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

    // subscribing to observables from within the vue component.
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
          type: 'select',
          id: 'selectfile',
          placeholder: 'Select file to append to',
          label: 'Select first file',
          options: [],
          value: '',
          noClose: true,
          title: 'Lists and allows selection from available files',
        },
        {
          type: 'source',
          id: 'sourcefile',
          placeholder: ' ',
          label: 'Source File Name',
          library: 'files',
          noClose: true,
          readonly: true,
          value: '',
          title: 'File name of file that will be exported',
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
          onClick: this.onClickAction.bind(this),
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
