<!-- Index file for the player route -->
<script lang="ts">
    import type {KeyID} from "src/types/devices";
    import {Color, ColorType} from "../types/color";

    import {virtualDeviceComponents} from "../components/devices/Devices";

    import type {Canvas, KeyPress, KeyRelease} from "../engine/CanvasAPI";
    import type {DeviceInfoCanvas, ProjectRT} from "../engine/ProjectRT";

    import {projectEngines} from "../engine/Engines";
    import {GridController} from "../hardware/hardware";

    import Information from "carbon-icons-svelte/lib/Information.svelte";

    import Popup from "../components/Popup.svelte";
    import Dropdown from "../components/Dropdown.svelte";
    import Sidebar from "../components/Sidebar.svelte";
    import Multibutton from "../components/Multibutton.svelte";
    import Switch from "../components/Switch.svelte";
    import MobileSidebarButton from "../components/MobileSidebarButton.svelte";
    import LayerSelector from "../components/LayerSelector.svelte";

    import ResizeObserver from "svelte-resize-observer";

    import {SvelteToast, toast} from "@zerodevx/svelte-toast";
    import {t, locale, locales} from "$lib/translations";

    import {browser} from "$app/environment";

    import {afterUpdate, onMount} from "svelte";
    import "../shared.css";

    const mobileViewWidthThreshold = 800;
    const mobileViewAspectThreshold = 3/5;

    const keyPressColors: { [key: string]: Color } = 
    {
        "color.green": new Color(ColorType.Palette, ["classic", 21]),
        "color.red": new Color(ColorType.Palette, ["classic", 5]),
        "color.blue": new Color(ColorType.Palette, ["classic", 45]),
        "color.cyan": new Color(ColorType.Palette, ["classic", 37]),
        "color.purple": new Color(ColorType.Palette, ["classic", 53]),
        "color.yellow": new Color(ColorType.Palette, ["classic", 13]),
        "color.white": new Color(ColorType.Palette, ["classic", 3])
    }

    // Settings are values that saves
    let settings = {
        virtualDevice: Object.keys(virtualDeviceComponents)[0],
        virtualDeviceScale: "100%",
        projectEngine: "Unipack", //Object.keys(projectEngines)[0],
        deviceInput: undefined,
        deviceOutput: undefined,
        deviceConfig: undefined,
        deviceSettingAdvanced: false,
        language: undefined,
        keypressColor: Object.keys(keyPressColors)[0],
    };

    // Options are not somethings being saved
    let options = 
    {
        lightAnimation: true,
        showKeyPress: false,
        learningMode: false
    }
    
    var player_ready = false;

    $: if (browser && player_ready) {
        console.log("Saving setting");
        localStorage.setItem("settings", JSON.stringify(settings));
    }

    let virtualDeviceComponent: typeof virtualDeviceComponents[number]["component"];
    $: virtualDeviceComponent = 
        virtualDeviceComponents[settings.virtualDevice] 
            ? virtualDeviceComponents[settings.virtualDevice].component 
            : virtualDeviceComponents["Mystrix"].component 


    let engine: ProjectRT;
    let projectStatus: string = "not loaded";

    let virtualDevices: any[] = []; //Should be fine
    let virtualDevicesInfo: DeviceInfoCanvas[] = [];

    let popup: { [key: string]: boolean } = {};

    let mobileView = false;
    let showSidebar = true;

    const updateDevicesInfo = () => {
        virtualDevicesInfo = [];
        virtualDevices.forEach((device) => {
            virtualDevicesInfo.push({
                id: device.id,
                pos: device.pos,
                info: device.deviceInfo,
            });
        });
    };

    const virtualKeyPressed: KeyPress = (deviceID: number, keyID: KeyID) => {
        console.info(`Virtual Device ${keyID} has been pressed`);
        // device.setColor(keyID, new Color(ColorType.RGB, [255, 255, 255]));
        // console.log(deviceInfo)
        engine?.KeyPress(virtualDevicesInfo[deviceID], keyID);
    };

    const virtualKeyReleased: KeyRelease = (deviceID: number, keyID: KeyID) => {
        console.info(`Virtual Device ${keyID} has been released`);
        // // device.setColor(keyID, new Color(ColorType.RGB, [0, 0, 0]));

        engine?.KeyRelease(virtualDevicesInfo[deviceID], keyID);
    };

    let midiDevices: GridController[] = [];
    let midiDeviceInfos: DeviceInfoCanvas = [];

    const deviceKeyPressed: KeyPress = (deviceID: number, keyID: KeyID) => {
        console.info(`Device ${deviceID} Button ${keyID} has been pressed`);
        // device.setColor(keyID, new Color(ColorType.RGB, [255, 255, 255]));
        // console.log(deviceInfo)
        engine?.KeyPress(midiDeviceInfos[deviceID], keyID);
    };

    const deviceKeyReleased: KeyRelease = (deviceID: number, keyID: KeyID) => {
        console.info(`Device ${deviceID} Button ${keyID} has been released`);
        // // device.setColor(keyID, new Color(ColorType.RGB, [0, 0, 0]));

        engine?.KeyRelease(midiDeviceInfos[deviceID], keyID);
    };

    const deviceEvent = (event: {}) => {
        console.log(`Midi Device Event`);
        console.log(event);
        switch (event.event) {
            case "opened":
                midiDeviceInfos[event.deviceID] = {
                    name: midiDevices[event.deviceID].name,
                    id: event.deviceID,
                    pos: [0, 0],
                    info: midiDevices[event.deviceID].getDeviceInfo(),
                };
                reactiveVars.activeDevice = midiDevices[event.deviceID]?.name;
                reactiveVars.activeInput =
                    midiDevices[event.deviceID]?.activeInput?.name;
                reactiveVars.activeOutput =
                    midiDevices[event.deviceID]?.activeOutput?.name;
                reactiveVars.activeConfig =
                    midiDevices[event.deviceID]?.activeConfig?.name;
                toast.push(
                    $t("toast.is_now_the_active_device", {
                        device_name: midiDeviceInfos[event.deviceID].name,
                    }),
                    {
                        theme: {
                            "--toastColor": "#FFFFFF;",
                            "--toastBackground": "#48BB78",
                            "--toastBarBackground": "#2F855A",
                        },
                    }
                );
                break;
            case "closed":
                if (midiDeviceInfos[event.deviceID] != undefined) {
                    //So when user action caused port close (it will set deviceInfo to undefined). No toast will be shown
                    toast.push(
                        $t("toast.no_longer_active", {
                            device_name: midiDeviceInfos[event.deviceID].name,
                        }),
                        {
                            theme: {
                                "--toastColor": "#FFFFFF;",
                                "--toastBackground": "#F56565",
                                "--toastBarBackground": "#C53030",
                            },
                        }
                    );
                }
                midiDeviceInfos[event.deviceID] = undefined;
                reactiveVars.activeDevice = undefined;
                reactiveVars.activeInput = undefined;
                reactiveVars.activeOutput = undefined;
                reactiveVars.activeConfig = undefined;
                break;

            case "connected":
                if (event.device == settings.deviceInput) {
                    toast.push(
                        $t("toast.connected", {device_name: event.device})
                    );
                    midiDevices[0].connect(
                        GridController.availableDeviceInputs()[event.device],
                        GridController.availableDeviceOutputs()[event.device],
                        settings.deviceConfig
                    );
                } else {
                    toast.push(
                        $t("toast.detected", {device_name: event.device})
                    );
                    // toast.push(`${event.device} connected\nClick to set it as the active device`);
                }
                break;

            case "disconnected":
                toast.push(
                    $t("toast.disconnected", {device_name: event.device})
                );
                break;
        }
    };

    const onKeyDown = (e: KeyboardEvent) => {
        if(e.repeat) return;

        var keyCode = e.keyCode;
        if(keyCode > 47 && keyCode < 58) // 0-9
        {
            var layer = keyCode == 48 ? 9 : keyCode - 49;
            engine?.LayerChange(layer);
            return
        }
        
        switch(keyCode) {
            case 32: // Space
            case 80: // P
                engine?.demoplay?.status === "PLAYING" ? engine?.demoplay?.Pause() : engine?.demoplay?.Start();
                break;
            case 37: // Left - Previous Action
            case 65: // A
                engine?.demoplay?.Pause();
                engine?.demoplay?.Previous();
                break;
            case 39: // Right - Next Action
            case 68: // D
                engine?.demoplay?.Pause();
                engine?.demoplay?.Next();
                break;
            case 38: // Up - Layer Down
            case 87: // W
            case 69: // E
                engine?.LayerChange(engine?.currentLayer + 1);
                break;
            case 40: // Down - Layer Up
            case 83: // S
            case 81: // Q
                engine?.LayerChange(engine?.currentLayer - 1);
                break;
            case 70: // F - Full Screen
            case 13: // Enter
            case 27: // Esc
                showSidebar = !showSidebar;
                break;
            case 82: // R - Reload
                loadProject();
                break;
            case 90: // Z - Show Settings
                if(popup["devices"] || popup["demoplay"]) 
                {
                    popup["devices"] = false;
                    popup["demoplay"] = false;
                }
                popup["setting"] = !popup["setting"];
                break;
            case 88: // X - Show Devices
                if(popup["setting"] || popup["demoplay"]) 
                {
                    popup["setting"] = false;
                    popup["demoplay"] = false;
                }
                popup["devices"] = !popup["devices"];
                break;
            case 67: // C - Show Demo Play Settings
                if(popup["setting"] || popup["devices"]) 
                {
                    popup["setting"] = false;
                    popup["devices"] = false;
                }
                popup["demoplay"] = !popup["demoplay"];
                break;
            default:
                break;
        }
	}

    midiDevices[0] = new GridController(0, deviceKeyPressed, deviceKeyReleased);

    const loadProject = () => {
        console.log("Load File Selector");
        var input = document.createElement("input");
        input.type = "file";
        input.accept = engine.fileFormat;
        input.onchange = (e) => {
            var file = e?.target?.files[0];
            engine.LoadProjectFile(file).then(
                (result) => {
                    console.log("Project Loaded");
                    projectStatus = "loaded";
                },
                (error) => {
                    toast.push(
                        $t("toast.project_failed_to_load", {project_loading_error: error.toString()}
                        ),
                        {
                            theme: {
                                "--toastColor": "#FFFFFF;",
                                "--toastBackground": "#F56565",
                                "--toastBarBackground": "#C53030",
                            },
                            duration: 5000
                        }
                    );
                    projectStatus = "not loaded";
                }
            );
            projectStatus = "loading";
        };
        input.click();
    };

    let overlays:any[] = [];

    var api: Canvas = {
        setColor: function (deviceID: number, keyID: KeyID, color: Color) {
            var signature = [deviceID, keyID];
            if(!overlays.map(String).includes(signature.toString()))
            {
                virtualDevices[deviceID].setColor(keyID, color);
                midiDevices[deviceID]?.setColor(keyID, color);
            }
        },

        setOverlay: function (deviceID: number, keyID: KeyID, color?: Color) {
            var signature = [deviceID, keyID];

            if(color === undefined)
            {
                color = keyPressColors[settings.keypressColor];
            }

            if(!color.isBlack())
            {
                if(!overlays.map(String).includes(signature.toString())) {overlays.push(signature);}
            }
            else
            {
                let index = overlays.map(String).indexOf(signature.toString())
                if(index != -1) {overlays.splice(index, 1);}
            }

            virtualDevices[deviceID].setColor(keyID, color);
            midiDevices[deviceID]?.setColor(keyID, color);
        },

        unsetOverlay(deviceID: number, keyID: KeyID)
        {
            var signature = [deviceID, keyID];
            let index = overlays.map(String).indexOf(signature.toString());
            if(index != -1) {overlays.splice(index, 1);}
            virtualDevices[deviceID].setColor(keyID, new Color(ColorType.Palette, ["classic", 0]));
            midiDevices[deviceID]?.setColor(keyID, new Color(ColorType.Palette, ["classic", 0]));
        },

        clearOverlay: function(targetDeviceID?: number){
            for(let overlay of overlays)
            {
                let [deviceID, keyID] = overlay;
                if(targetDeviceID === undefined || deviceID == targetDeviceID)
                {
                    virtualDevices[deviceID].setColor(keyID, new Color(ColorType.Palette, ["classic", 0]));
                    midiDevices[deviceID]?.setColor(keyID, new Color(ColorType.Palette, ["classic", 0]));
                }
            }
            overlays = [];
        },

        clear: function (deviceID?: number) {
            // virtualDevices[deviceID].clear(); //TODO: Implentment this
            // midiDevices[deviceID].clear(); //TODO: Implentment this
        },

        getDevices: function () {
            return virtualDevicesInfo;
        },

        options: options
    };

    let reactiveVars = {
        activeDevice: undefined,
        activeInput: undefined,
        activeOutput: undefined,
        activeConfig: undefined,
    };

    if (browser) {
        if (localStorage.getItem("settings")) {
            settings = JSON.parse(localStorage.getItem("settings")!);

            GridController.start(deviceEvent).then((midi_available) => {
                if (midi_available) {
                    toast.push(
                        $t("toast.webmidi_available"),
                        {
                            theme: {
                                "--toastColor": "#FFFFFF;",
                                "--toastBackground": "#48BB78",
                                "--toastBarBackground": "#2F855A",
                            },
                        }
                    );
                    midiDevices[0].connect(
                        GridController.availableDeviceInputs()[settings.deviceInput!],
                        GridController.availableDeviceOutputs()[settings.deviceOutput!],
                        GridController.configList()[settings.deviceConfig!]
                    );
                } else {
                    toast.push(
                        $t("toast.webmidi_unavailable"),
                        {
                            theme: {
                                "--toastColor": "#FFFFFF;",
                                "--toastBackground": "#F56565",
                                "--toastBarBackground": "#C53030",
                            },
                            duration: 10000
                        }
                    );
                }
            })

            // console.log(settings);
        }

        player_ready = true;
        if(projectEngines[settings.projectEngine] === undefined)
        {
            settings.projectEngine = Object.keys(projectEngines)[0]; //Revert to the first one
        }
        engine = projectEngines[settings.projectEngine](api);
    }

    onMount(() => {
            setInterval(() => {
                if (reactiveVars.activeConfig != midiDevices[0]?.activeConfig?.name)
                    reactiveVars.activeConfig = midiDevices[0]?.activeConfig?.name;
            }, 1000 / 30);
        }
    );

    afterUpdate(() => {
        updateDevicesInfo();
    });

    function onResize(e) {
        let width = e.detail.clientWidth;
        let height = e.detail.clientHeight;
        let newMobileView = width/height <= mobileViewAspectThreshold || width < mobileViewWidthThreshold;

        if(newMobileView != mobileView)
        {
            mobileView = newMobileView;
        }
    }
</script>

<main>
    <div class="toast {mobileView ? 'mobile': ''}">
        <SvelteToast options={{pausable: true, intro: mobileView ? { y: -192 } : undefined}}/>
    </div>
    {#if player_ready}
        {#if mobileView}
            <div class="mobile-header">
                <div class="amethyst-bar center-class">
                    <img src="logo-256.png">

                    <div style="margin-left: 10px">
                        <span>Amethyst</span>
                    </div>
                </div>

                <div class="show-controls-icon-parent center-class">
                    <MobileSidebarButton bind:checked={showSidebar}/>
                </div>
            </div>
        {/if}
        <div class="main-content">
            <ResizeObserver on:resize={onResize} />
            <Sidebar
                    on:settings={() => (popup["setting"] = true)}
                    on:devices={() => (popup["devices"] = true)}
                    on:demoplay={() => (popup["demoplay"] = true)}
                    on:loadProject={() => {loadProject();}}
                    bind:project={engine}
                    bind:status={projectStatus}
                    bind:show={showSidebar}
                    bind:mobile={mobileView}
            />

            <div class="content-part" on:click={(e) => {if(!mobileView)showSidebar = !showSidebar}}>
                <div class="amethyst-player-content {mobileView ? 'mobile' : ''}" >
                    <div class="amethyst-player-launchpad-holder center-class">
                        <div
                                style={`width: 85%; max-width: ${50 * parseInt(settings.virtualDeviceScale) / 100}vh;`}
                                class="center-class"
                                on:click={(e) => {e.stopPropagation()}}
                        >
                                <svelte:component
                                        this={virtualDeviceComponent}
                                        bind:this={virtualDevices[0]}
                                        id={0}
                                        pos={[0, 0]}
                                        keyPress={virtualKeyPressed}
                                        keyRelease={virtualKeyReleased}
                                />
                        </div>
                    </div>
                </div>
                <div class="amethyst-player-footer center-class" on:click={(e) => {e.stopPropagation()}}>
                {#if projectStatus === "loaded"}
                    <LayerSelector bind:project={engine}/>
                {:else}
                    <span title= {__BUILD_STRING__}>
                        {`Amethyst Player`}
                    </span>
                {/if}
                </div>
            </div>
        </div>
    {/if}

    <Popup bind:mobile={mobileView} bind:show={popup["setting"]}>
        <div class="settings-popup">
            <div class="popup-header center-class">
                <span>{$t("setting.settings")}</span>
            </div>

            <div class="setting {mobileView? 'mobile' : ''}">
                <div class="setting-name">
                    <span>{$t("setting.virtual_device") + ":"}</span>
                </div>

                <div class="setting-option">
                    <Dropdown
                            bind:value={settings.virtualDevice}
                            options={Object.keys(virtualDeviceComponents)}
                    />
                </div>
            </div>

            <div class="setting {mobileView? 'mobile' : ''}">
                <div class="setting-name">
                    <span>{$t("setting.virtual_device_Scale") + ":"}</span>
                </div>

                <div class="setting-option">
                    <Multibutton
                            bind:value={settings.virtualDeviceScale}
                            options={["50%", "75%", "100%", "125%", "150%"]}
                    />
                </div>
            </div>

            <div class="setting {mobileView? 'mobile' : ''}">
                <div class="setting-name">
                    <span>{$t("setting.project_engine") + ":"}</span>
                </div>

                <div class="setting-option">
                    <Dropdown
                        bind:value={settings.projectEngine}
                        options={Object.keys(projectEngines)}
                        on:change={() => {
                        engine =
                            projectEngines[settings.projectEngine](api);
                        }}
                    />
                </div>
            </div>

            <div class="setting {mobileView? 'mobile' : ''}">
                <div class="setting-name">
                    <span>{$t("setting.language") + ":"}</span>
                </div>

                <div class="setting-option">
                    <Dropdown
                        value={$t(`lang.${locale.get()}`)}
                        options={$locales.map((x) =>
                            $t(`lang.${x}`)
                        )}
                        on:change={(e) => {
                        $locale = $locales[e.detail.index];
                        settings.language = $locales[e.detail.index];
                        }}
                    />
                </div>
            </div>
        </div>
    </Popup>

    <Popup bind:mobile={mobileView} bind:show={popup["devices"]}>
        <div class="settings-popup">
            <div class="popup-header center-class">
                <span>{$t("device.device")}</span>
            </div>

            {#if !settings.deviceSettingAdvanced}
                <div class="setting {mobileView? 'mobile' : ''}">
                    <div class="setting-name">
                        <span>{$t("device.midi_device") + ":"}</span>
                    </div>

                    <div class="setting-option">
                        <Dropdown
                                value={reactiveVars.activeDevice}
                                options={Object.keys(
                                GridController.availableDevices()
                            )}
                                placeholder={$t("device.no_device")}
                                on:change={(e) => {
                                settings.deviceInput = e.detail.value;
                                settings.deviceOutput = e.detail.value;
                                if (e.detail.value) {
                                    midiDeviceInfos[0] = undefined;
                                    midiDevices[0].connectDevice(
                                        GridController.availableDevices()[
                                            e.detail.value
                                        ]
                                    );
                                } else {
                                    midiDevices[0].disconnect();
                                }
                            }}
                        />
                    </div>
                </div>
            {:else}
                <div class="setting {mobileView? 'mobile' : ''}">
                    <div class="setting-name">
                        <span>{$t("device.midi_input_device") + ":"}</span>
                    </div>

                    <div class="setting-option">
                        <Dropdown
                                value={reactiveVars.activeInput}
                                options={Object.keys(
                                GridController.availableDeviceInputs()
                            )}
                                placeholder={$t("device.no_device")}
                                on:change={(e) => {
                                settings.deviceInput = e.detail.value;
                                midiDeviceInfos[0] = undefined;
                                midiDevices[0].connect(
                                    GridController.availableDeviceInputs()[
                                        e.detail.value
                                    ],
                                    midiDevices[0].activeOutput,
                                    midiDevices[0].activeConfig
                                );
                            }}
                        />
                    </div>
                </div>

                <div class="setting {mobileView? 'mobile' : ''}">
                    <div class="setting-name">
                        <span>{$t("device.midi_output_device") + ":"}</span>
                    </div>

                    <div class="setting-option">
                        <Dropdown
                                value={reactiveVars.activeOutput}
                                options={Object.keys(
                                GridController.availableDeviceOutputs()
                            )}
                                placeholder={$t("device.no_device")}
                                on:change={(e) => {
                                settings.deviceOutput = e.detail.value;
                                midiDeviceInfos[0] = undefined;
                                midiDevices[0].connect(
                                    midiDevices[0].activeInput,
                                    GridController.availableDeviceOutputs()[
                                        e.detail.value
                                    ],
                                    midiDevices[0].activeConfig
                                );
                            }}
                        />
                    </div>
                </div>
            {/if}

            <div class="setting {mobileView? 'mobile' : ''}">
                <div class="setting-name">
                    <span>{$t("device.midi_device_config") + ":"}</span>
                </div>

                <div class="setting-option">
                    <Dropdown
                            value={reactiveVars.activeConfig}
                            options={Object.keys(GridController.configList())}
                            placeholder={$t("device.no_config")}
                            on:change={(e) => {
                            settings.deviceConfig = e.detail.value;
                            if (e.detail.value) {
                                midiDeviceInfos[0] = undefined;
                                midiDevices[0].connect(
                                    midiDevices[0].activeInput,
                                    midiDevices[0].activeOutput,
                                    GridController.configList()[e.detail.value]
                                );
                            } else {
                                midiDevices[0].disconnect();
                            }
                        }}
                    />
                </div>
            </div>

            <div class="setting {mobileView? 'mobile' : ''}">
                <div class="setting-name">
                    <span>{$t("device.advanced_mode") + ":"}</span>
                </div>

                <div class="setting-option">
                    <Switch bind:checked={settings.deviceSettingAdvanced}/>
                </div>
            </div>
        </div>
    </Popup>

    <Popup bind:mobile={mobileView} bind:show={popup["demoplay"]}>
        <div class="settings-popup">
            <div class="popup-header center-class">
                <span>{$t("demoplay.demoplay")}</span>
            </div>

            <div class="setting {mobileView? 'mobile' : ''}">
                <div class="setting-name">
                    <span>{$t("demoplay.light_animation")}</span>
                </div>

                <div class="setting-option">
                    <Switch bind:checked={options.lightAnimation}/>
                </div>
            </div>

            <div class="setting {mobileView? 'mobile' : ''}">
                <div class="setting-name">
                    <span>{$t("demoplay.show_key_press")}</span>
                </div>

                <div class="setting-option">
                    <Switch bind:checked={options.showKeyPress}
                    on:change={(e) => {
                        if(!e.detail.checked){options.learningMode = false; api.clearOverlay();}
                        else { engine.demoplay?.showActionKeys();} //NOTE THIS IS NOT A STANDARD PROJECTRT API}
                    }} 
                    />
                </div>
            </div>

            <div class="setting {mobileView? 'mobile' : ''}">
                <div class="setting-name">
                    <span>{$t("demoplay.key_press_color")}</span>
                </div>

                <div class="setting-option">
                    <Dropdown
                        bind:value={settings.keypressColor}
                        options={Object.keys(keyPressColors)}
                    />
                </div>
            </div>


            <div class="setting {mobileView? 'mobile' : ''}">
                <div
                        class="setting-name"
                        title={$t("demoplay.learning_mode_info")}
                >
                    <span>{$t("demoplay.learning_mode")}</span>
                    <Information
                            size={14}
                            style="margin-left: 4px; margin-top: 2px; color:#A0A0A0;"
                    />
                </div>

                <div class="setting-option">
                    <Switch bind:checked={options.learningMode}
                        on:change={(e) => {
                            if(e.detail.checked){
                                options.showKeyPress = true;
                                engine.demoplay?.showActionKeys() //NOTE THIS IS NOT A STANDARD PROJECTRT API
                            }
                        }} 
                    />
                </div>
            </div>
        </div>
    </Popup>
</main>

<style lang="scss">
    main {
        background-color: #1d1d1d;
        height: 100vh;
        width: 100vw;

        overflow: hidden;
    }

    .center-class {
        display: flex;
        justify-content: center;
        align-items: center;
    }

    .main-content {
        display: flex;
        flex-direction: row;
        justify-content: space-between;

        .content-part {
            width: 100%;
            max-width: 100vw;
        }
    }

    .amethyst-player-content {
        height: calc(100vh - 100px);
        width: 100%;

        overflow: auto;

        &.mobile{
            height: calc(100vh - 160px);
        }

        .amethyst-player-launchpad-holder {
            height: calc(100%);
            overflow: hidden;

            .circular-loader-bottom-text {
                font-family: 'Roboto', sans-serif;
                font-style: normal;
                font-weight: 300;
                font-size: 20px;
                color: rgba(245, 245, 245, 0.52);
            }
        }
    }

    .amethyst-player-footer {
        height: 100px;
        width: 100%;
        flex-direction: column;
        gap: 10px;

        span {
            font-size: 18px;

            font-family: "Roboto Mono", sans-serif;
            color: #969696;
            font-weight: 300;

            a {
                color: aqua;

                &:hover {
                    color: #0ec0c0;
                }
            }
        }
    }

    .settings-popup {
        display: flex;
        flex-direction: column;
        gap: 10px;

        .popup-header {
            height: 30px;
            font-size: 26px;

            font-family: "Roboto Mono", sans-serif;
            color: whitesmoke;
            font-weight: 300;

            margin-bottom: 20px;
        }

        .setting {
            height: 35px;

            display: flex;
            align-items: center;

            .setting-name {
                width: 250px;
                display: flex;
                align-items: center;

                color: whitesmoke;

                font-family: "Roboto Mono", sans-serif;
                font-weight: 400;
            }

            .setting-option {
                min-width: 300px;
                display: flex;
                flex-direction: row-reverse;
            }

            &.mobile {
                flex-direction: column;
                height: auto;
                gap: 10px;

                .setting-name {
                    width: 100%;
                }

                .setting-option {
                    width: 100%;
                    flex-direction: row;
                }
            }
        }
    }

    .toast {
        display: contents;
        font-family: "Roboto Mono", sans-serif;
        font-style: normal;
        font-size: 16px;
        font-weight: 300;
        --toastWidth: 20rem;
        --toastColor: #cbcbcb;
        --toastBackground: #141414;
        --toastBarBackground: #3e3e3e;

        &.mobile
        {
            --toastWidth: 100vw;
            --toastHeight: 4rem;
            --toastContainerTop: 0;
            --toastContainerLeft: 0;
        }
    }

    .mobile-header {
        height: 60px;

        background-color: rgb(20, 20, 20);

        .show-controls-icon-parent {
            position: fixed;

            height: 60px;
            width: 60px;
        }

        .amethyst-bar {
            position: fixed;

            height: 60px;
            width: 100%;

            img {
                height: 52px;
            }

            span {
                font-family: 'Roboto', sans-serif;
                font-style: normal;
                font-weight: 300;
                font-size: 22px;

                letter-spacing: 0.125rem;
                color: #f5f5f5;
            }
        }
    }
</style>

<svelte:window on:keydown|preventDefault={onKeyDown} />
