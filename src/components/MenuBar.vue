<template>
  <q-bar class="bg-background q-pa-none relative-position">
    <div
      v-if="$q.platform.is.mac && !isFullscreen"
      class="mac-traffic-light-space"
    ></div>
    <img v-else src="icon.png" class="window-logo" alt="application logo" />
    <menu-button
      v-for="(root, index) of menudata"
      :key="index"
      :menudata="root"
      v-model:selected="subMenuOpenFlags[index]"
      :disable="menubarLocked"
      @mouseover="reassignSubMenuOpen(index)"
      @mouseleave="
        root.type === 'button' ? (subMenuOpenFlags[index] = false) : undefined
      "
    />
    <q-space />
    <div class="window-title">
      {{
        (isEdited ? "*" : "") +
        (projectName !== undefined ? projectName + " - " : "") +
        "VOICEVOX" +
        (currentVersion ? " - Ver. " + currentVersion + " - " : "") +
        (useGpu ? "GPU" : "CPU")
      }}
    </div>
    <q-space />
    <title-bar-buttons />
  </q-bar>
</template>

<script lang="ts">
import { defineComponent, ref, computed, ComputedRef, watch } from "vue";
import { useStore } from "@/store";
import MenuButton from "@/components/MenuButton.vue";
import TitleBarButtons from "@/components/TitleBarButtons.vue";
import { useQuasar } from "quasar";
import { HotkeyAction, HotkeyReturnType } from "@/type/preload";
import { setHotkeyFunctions } from "@/store/setting";
import {
  generateAndConnectAndSaveAudioWithDialog,
  generateAndSaveAllAudioWithDialog,
  generateAndSaveOneAudioWithDialog,
  connectAndExportTextWithDialog,
} from "@/components/Dialog";

type MenuItemBase<T extends string> = {
  type: T;
  label?: string;
};

export type MenuItemSeparator = MenuItemBase<"separator">;

export type MenuItemRoot = MenuItemBase<"root"> & {
  onClick: () => void;
  subMenu: MenuItemData[];
};

export type MenuItemButton = MenuItemBase<"button"> & {
  onClick: () => void;
};

export type MenuItemCheckbox = MenuItemBase<"checkbox"> & {
  checked: ComputedRef<boolean>;
  onClick: () => void;
};

export type MenuItemData =
  | MenuItemSeparator
  | MenuItemRoot
  | MenuItemButton
  | MenuItemCheckbox;

export type MenuItemType = MenuItemData["type"];

export default defineComponent({
  name: "MenuBar",

  components: {
    MenuButton,
    TitleBarButtons,
  },

  setup() {
    const store = useStore();
    const $q = useQuasar();
    const currentVersion = ref("");
    window.electron.getAppInfos().then((obj) => {
      currentVersion.value = obj.version;
    });
    const uiLocked = computed(() => store.getters.UI_LOCKED);
    const menubarLocked = computed(() => store.getters.MENUBAR_LOCKED);
    const projectName = computed(() => store.getters.PROJECT_NAME);
    const useGpu = computed(() => store.state.useGpu);
    const isEdited = computed(() => store.getters.IS_EDITED);
    const isFullscreen = computed(() => store.getters.IS_FULLSCREEN);

    const createNewProject = async () => {
      if (!uiLocked.value) {
        await store.dispatch("CREATE_NEW_PROJECT", {});
      }
    };

    const generateAndSaveAllAudio = async () => {
      if (!uiLocked.value) {
        await generateAndSaveAllAudioWithDialog({
          encoding: store.state.savingSetting.fileEncoding,
          quasarDialog: $q.dialog,
          dispatch: store.dispatch,
        });
      }
    };

    const generateAndConnectAndSaveAllAudio = async () => {
      if (!uiLocked.value) {
        await generateAndConnectAndSaveAudioWithDialog({
          quasarDialog: $q.dialog,
          dispatch: store.dispatch,
          encoding: store.state.savingSetting.fileEncoding,
        });
      }
    };

    const generateAndSaveOneAudio = async () => {
      if (uiLocked.value) return;

      const activeAudioKey = store.getters.ACTIVE_AUDIO_KEY;
      if (activeAudioKey == undefined) {
        $q.dialog({
          title: "?????????????????????????????????????????????",
          message: "????????????????????????????????????????????????????????????????????????",
          ok: {
            label: "?????????",
            flat: true,
            textColor: "secondary",
          },
        });
        return;
      }

      await generateAndSaveOneAudioWithDialog({
        audioKey: activeAudioKey,
        encoding: store.state.savingSetting.fileEncoding,
        quasarDialog: $q.dialog,
        dispatch: store.dispatch,
      });
    };

    const connectAndExportText = async () => {
      if (!uiLocked.value) {
        await connectAndExportTextWithDialog({
          quasarDialog: $q.dialog,
          dispatch: store.dispatch,
          encoding: store.state.savingSetting.fileEncoding,
        });
      }
    };

    const importTextFile = () => {
      if (!uiLocked.value) {
        store.dispatch("COMMAND_IMPORT_FROM_FILE", {});
      }
    };

    const saveProject = () => {
      if (!uiLocked.value) {
        store.dispatch("SAVE_PROJECT_FILE", { overwrite: true });
      }
    };

    const saveProjectAs = () => {
      if (!uiLocked.value) {
        store.dispatch("SAVE_PROJECT_FILE", {});
      }
    };

    const importProject = () => {
      if (!uiLocked.value) {
        store.dispatch("LOAD_PROJECT_FILE", {});
      }
    };
    const closeAllDialog = () => {
      store.dispatch("IS_SETTING_DIALOG_OPEN", {
        isSettingDialogOpen: false,
      });
      store.dispatch("IS_HELP_DIALOG_OPEN", {
        isHelpDialogOpen: false,
      });
      store.dispatch("IS_HOTKEY_SETTING_DIALOG_OPEN", {
        isHotkeySettingDialogOpen: false,
      });
      store.dispatch("IS_TOOLBAR_SETTING_DIALOG_OPEN", {
        isToolbarSettingDialogOpen: false,
      });
      store.dispatch("IS_CHARACTER_ORDER_DIALOG_OPEN", {
        isCharacterOrderDialogOpen: false,
      });
      store.dispatch("IS_DEFAULT_STYLE_SELECT_DIALOG_OPEN", {
        isDefaultStyleSelectDialogOpen: false,
      });
    };

    const openHelpDialog = () => {
      store.dispatch("IS_HELP_DIALOG_OPEN", {
        isHelpDialogOpen: true,
      });
    };

    const menudata = ref<MenuItemData[]>([
      {
        type: "root",
        label: "????????????",
        onClick: () => {
          closeAllDialog();
        },
        subMenu: [
          {
            type: "button",
            label: "??????????????????",
            onClick: () => {
              generateAndSaveAllAudio();
            },
          },
          {
            type: "button",
            label: "????????????????????????",
            onClick: () => {
              generateAndSaveOneAudio();
            },
          },
          {
            type: "button",
            label: "??????????????????????????????",
            onClick: () => {
              generateAndConnectAndSaveAllAudio();
            },
          },
          { type: "separator" },
          {
            type: "button",
            label: "????????????????????????????????????",
            onClick: () => {
              connectAndExportText();
            },
          },
          {
            type: "button",
            label: "????????????????????????",
            onClick: () => {
              importTextFile();
            },
          },
          { type: "separator" },
          {
            type: "button",
            label: "????????????????????????",
            onClick: createNewProject,
          },
          {
            type: "button",
            label: "????????????????????????????????????",
            onClick: () => {
              saveProject();
            },
          },
          {
            type: "button",
            label: "?????????????????????????????????????????????",
            onClick: () => {
              saveProjectAs();
            },
          },
          {
            type: "button",
            label: "??????????????????????????????",
            onClick: () => {
              importProject();
            },
          },
        ],
      },
      {
        type: "root",
        label: "????????????",
        onClick: () => {
          closeAllDialog();
        },
        subMenu: [
          {
            type: "button",
            label: "?????????",
            onClick: () => {
              store.dispatch("RESTART_ENGINE_ALL");
            },
          },
        ],
      },
      {
        type: "root",
        label: "??????",
        onClick: () => {
          closeAllDialog();
        },
        subMenu: [
          {
            type: "button",
            label: "??????????????????",
            onClick() {
              store.dispatch("IS_HOTKEY_SETTING_DIALOG_OPEN", {
                isHotkeySettingDialogOpen: true,
              });
            },
          },
          {
            type: "button",
            label: "????????????????????????????????????",
            onClick() {
              store.dispatch("IS_TOOLBAR_SETTING_DIALOG_OPEN", {
                isToolbarSettingDialogOpen: true,
              });
            },
          },
          {
            type: "button",
            label: "???????????????????????????????????????",
            onClick() {
              store.dispatch("IS_CHARACTER_ORDER_DIALOG_OPEN", {
                isCharacterOrderDialogOpen: true,
              });
            },
          },
          {
            type: "button",
            label: "???????????????????????????",
            onClick() {
              store.dispatch("IS_DEFAULT_STYLE_SELECT_DIALOG_OPEN", {
                isDefaultStyleSelectDialogOpen: true,
              });
            },
          },
          {
            type: "button",
            label: "?????????????????????????????????",
            onClick() {
              store.dispatch("IS_DICTIONARY_MANAGE_DIALOG_OPEN", {
                isDictionaryManageDialogOpen: true,
              });
            },
          },
          { type: "separator" },
          {
            type: "button",
            label: "???????????????",
            onClick() {
              store.dispatch("IS_SETTING_DIALOG_OPEN", {
                isSettingDialogOpen: true,
              });
            },
          },
        ],
      },
      {
        type: "button",
        label: "?????????",
        onClick: () => {
          if (store.state.isHelpDialogOpen) closeAllDialog();
          else {
            closeAllDialog();
            openHelpDialog();
          }
        },
      },
    ]);

    const subMenuOpenFlags = ref(
      [...Array(menudata.value.length)].map(() => false)
    );

    const reassignSubMenuOpen = (idx: number) => {
      if (subMenuOpenFlags.value[idx]) return;
      if (subMenuOpenFlags.value.find((x) => x)) {
        const arr = [...Array(menudata.value.length)].map(() => false);
        arr[idx] = true;
        subMenuOpenFlags.value = arr;
      }
    };

    const hotkeyMap = new Map<HotkeyAction, () => HotkeyReturnType>([
      ["????????????????????????", createNewProject],
      ["??????????????????", generateAndSaveAllAudio],
      ["????????????????????????", generateAndSaveOneAudio],
      ["??????????????????????????????", generateAndConnectAndSaveAllAudio],
      ["????????????????????????", importTextFile],
      ["????????????????????????????????????", saveProject],
      ["?????????????????????????????????????????????", saveProjectAs],
      ["??????????????????????????????", importProject],
    ]);

    setHotkeyFunctions(hotkeyMap);

    watch(uiLocked, () => {
      // UI??????????????????????????????????????????????????????????????????????????????????????????
      if (uiLocked.value) {
        subMenuOpenFlags.value = [...Array(menudata.value.length)].map(
          () => false
        );
      }
    });

    return {
      currentVersion,
      uiLocked,
      menubarLocked,
      projectName,
      isEdited,
      isFullscreen,
      subMenuOpenFlags,
      reassignSubMenuOpen,
      menudata,
      useGpu,
    };
  },
});
</script>

<style lang="scss">
@use '@/styles/colors' as colors;

.active-menu {
  background-color: rgba(colors.$primary-rgb, 0.3) !important;
}
</style>

<style scoped lang="scss">
@use '@/styles/variables' as vars;
@use '@/styles/colors' as colors;

.q-bar {
  min-height: vars.$menubar-height;
  -webkit-app-region: drag;
  > .q-btn {
    margin-left: 0;
    -webkit-app-region: no-drag;
  }
}

.window-logo {
  height: vars.$menubar-height;
}

.window-title {
  height: vars.$menubar-height;
  margin-right: 10%;
  text-overflow: ellipsis;
  overflow: hidden;
}

.mac-traffic-light-space {
  margin-right: 70px;
}
</style>
