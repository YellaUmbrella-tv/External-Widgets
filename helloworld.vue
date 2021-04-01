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
    append: function (e, tag1, tag2) {
        // Using the confirm dialog, you can check if the user confirms the action before executing
        // If the result of the dialog is Yes, the callback continues and the file is manipulated.
        this.launchConfirmDialog('Would you like to continue?', () => {
            this.translating = 'Appending ' + tag2 + ' onto ' + tag1;

            // Using the subtitles service we can get the metadata for the files in the project.
            var tl1 = this.appScope.subtitlesService.timelines[tag1];
            var tl2 = this.appScope.subtitlesService.timelines[tag2];

            // Content contains the title ids and beginning and end times.
            var content1 = tl1.content;
            var content2 = tl2.content;

            // Get the id of the final title in the first file.
            var insertafter = content1[content1.length - 1].i;

            // You can log information to be viewed in the chrome developer tools.
            console.log('insert after ' + insertafter);

            var after = content1.length - 1;

            var ids = [];
            for (var i = 0; i < content2.length; i++) {
                ids.push(content2[i].i);
            }

            // Using the subtitles service getContentText function you can get the text content of a title using their id(s).
            res = this.appScope.subtitlesService.getContentText(tag2, ids);

            // You can subscribe to the result of that function to access the data.
            res.subscribe({
                // different functions can be called through the subscription.
                // On next, data is sent along to the subscribed.
                next: content => {
                    // Looping through the results of getContentText function you can access each titles string text.
                    for (var i = 0; i < content.results.length; i++) {
                        // Using the subtitles service you can insert titles into a file using it's tag.
                        this.appScope.subtitlesService.insertContentText(
                            tag1, // The file we want to add into
                            content2[i].b, // The beginning of the title from the appending file
                            content2[i].e, // The end of the title from the appending file
                            after, // The title id, the last id from the file we're adding into.
                            content.results[i].content, // The string content from the file we've read.
                        );
                        // Set the new 'last' id.
                        after = content1.length - 1;
                    }
                },
                // The error function is sent when there's an error, you can react to this and post logs
                error: err => console.error('append got an error: ' + err),
                // Once the function has succeeded the complete function is sent.
                complete: () => console.log('append complete notification'),
            });
        })
    },

    // get list of available timelines
    loadTagList: function () {
      var tltags = Object.keys(this.appScope.subtitlesService.timelines);
      // The projects files are stored locally.
      this.subslist = [];
      for (var i = 0; i < tltags.length; i++) {
        var tl = this.appScope.subtitlesService.timelines[tltags[i]];
        if (tl.metadata && tl.metadata.text) {
          this.subslist.push(tltags[i]);
        }
      }
      // populate the select boxes with the appropriate tags
      if (this.subslist.length) {
        this.selectedtag = this.subslist[0];
        this.selectedtag2 = this.subslist[1];
      } else {
        this.selectedtag = '';
        this.selectedtag2 = '';
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

    onClickAlign: function () {
      this.selectedtag = this.menuX(this.dialogContent, 'selectfile', 'value');
      this.selectedtag2 = this.menuX(this.dialogContent, 'selectfile2', 'value');
      if (this.selectedtag && this.selectedtag2 && this.selectedtag !== this.selectedtag2) {
        this.append(null, this.selectedtag, this.selectedtag2);
      }
    },

    // A custom function that runs when the dialog opens.
    onLaunchCustomDialog: function (item) {
      this.loadTagList();
      if (item.parent && item.parent.data) {
        this.selectedtag = item.parent.data.timelineid;
      }

      // Using menuX you can populate dialog fields with values loaded from the project.
      this.menuX(this.dialogContent, 'selectfile', 'options', this.subslist);
      this.menuX(this.dialogContent, 'selectfile', 'value', this.selectedtag);
      if (
        (this.subslist.length > 0 && !this.selectedtag) ||
        (this.subslist.length > 0 && this.subslist.indexOf(this.selectedtag) < 0)
      ) {
        this.selectedtag = this.subslist[0];
        this.getDefaults();
        this.menuSet('selectfile', 'value', this.subslist[0]);
      }
      this.menuX(this.dialogContent, 'selectfile2', 'options', this.subslist);
      this.menuX(this.dialogContent, 'selectfile2', 'value', this.selectedtag2);
      if (
        (this.subslist.length > 1 && !this.selectedtag2) ||
        (this.subslist.length > 1 && this.subslist.indexOf(this.selectedtag2) < 0)
      ) {
        this.selectedtag2 = this.subslist[1];
        this.getDefaults();
        this.menuSet('selectfile2', 'value', this.subslist[1]);
      }

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
              this.loadTagList();
              break;
          }
        }
      }
    });
    this.loadTagList();

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

    // Getting a unique idea by using the id of this vue element.
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
          placeholder: 'Select first file.',
          label: 'Select file to append to',
          options: [],
          value: '',
          noClose: true,
          title: 'Lists and allows selection from available files',
        },
        {
          type: 'select',
          id: 'selectfile2',
          placeholder: 'Select file to append to the above',
          label: 'Select second file',
          options: [],
          value: '',
          noClose: true,
          title: 'Lists and allows selection from available files',
        },
        {
          type: 'button',
          id: 'render',
          label: 'Append',
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
