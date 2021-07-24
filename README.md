module.exports = (_ => {
	const config = {
		"info": {
			"name": "ShowHiddenChannels",
			"author": "DevilBro",
			"version": "2.9.3",
			"description": "Display channels that are hidden from you by role restrictions"
		},
		"changeLog": {
			"added": {
				"Hide connected voice users": "You can now disable the option to show users who are in a hidden voice channel"
			}
		}
	};

	return !window.BDFDB_Global || (!window.BDFDB_Global.loaded && !window.BDFDB_Global.started) ? class {
		getName () {return config.info.name;}
		getAuthor () {return config.info.author;}
		getVersion () {return config.info.version;}
		getDescription () {return `The Library Plugin needed for ${config.info.name} is missing. Open the Plugin Settings to download it. \n\n${config.info.description}`;}
		
		downloadLibrary () {
			require("request").get("https://mwittrien.github.io/BetterDiscordAddons/Library/0BDFDB.plugin.js", (e, r, b) => {
				if (!e && b && b.indexOf(`* @name BDFDB`) > -1) require("fs").writeFile(require("path").join(BdApi.Plugins.folder, "0BDFDB.plugin.js"), b, _ => BdApi.showToast("Finished downloading BDFDB Library", {type: "success"}));
				else BdApi.alert("Error", "Could not download BDFDB Library Plugin, try again later or download it manually from GitHub: https://github.com/mwittrien/BetterDiscordAddons/tree/master/Library/");
			});
		}
		
		load () {
			if (!window.BDFDB_Global || !Array.isArray(window.BDFDB_Global.pluginQueue)) window.BDFDB_Global = Object.assign({}, window.BDFDB_Global, {pluginQueue: []});
			if (!window.BDFDB_Global.downloadModal) {
				window.BDFDB_Global.downloadModal = true;
				BdApi.showConfirmationModal("Library Missing", `The Library Plugin needed for ${config.info.name} is missing. Please click "Download Now" to install it.`, {
					confirmText: "Download Now",
					cancelText: "Cancel",
					onCancel: _ => {delete window.BDFDB_Global.downloadModal;},
					onConfirm: _ => {
						delete window.BDFDB_Global.downloadModal;
						this.downloadLibrary();
					}
				});
			}
			if (!window.BDFDB_Global.pluginQueue.includes(config.info.name)) window.BDFDB_Global.pluginQueue.push(config.info.name);
		}
		start () {this.load();}
		stop () {}
		getSettingsPanel () {
			let template = document.createElement("template");
			template.innerHTML = `<div style="color: var(--header-primary); font-size: 16px; font-weight: 300; white-space: pre; line-height: 22px;">The Library Plugin needed for ${config.info.name} is missing.\nPlease click <a style="font-weight: 500;">Download Now</a> to install it.</div>`;
			template.content.firstElementChild.querySelector("a").addEventListener("click", this.downloadLibrary);
			return template.content.firstElementChild;
		}
	} : (([Plugin, BDFDB]) => {
		var blackList = [], collapseList = [], hiddenCategory, lastGuildId, overrideTypes = [];
		var hiddenChannelCache = {};
		var settings = {};
		
		const settingsMap = {
			GUILD_TEXT: "showText",
			GUILD_VOICE: "showVoice",
			GUILD_ANNOUNCEMENT: "showAnnouncement",
			GUILD_STORE: "showStore"
		};

		const typeNameMap = {
			GUILD_TEXT: "TEXT_CHANNEL",
			GUILD_VOICE: "VOICE_CHANNEL",
			GUILD_ANNOUNCEMENT: "NEWS_CHANNEL",
			GUILD_STORE: "STORE_CHANNEL",
			GUILD_CATEGORY: "CATEGORY"
		};

		const channelIcons = {
			GUILD_TEXT: `M 5.88657 21 C 5.57547 21 5.3399 20.7189 5.39427 20.4126 L 6.00001 17 H 2.59511 C 2.28449 17 2.04905 16.7198 2.10259 16.4138 L 2.27759 15.4138 C 2.31946 15.1746 2.52722 15 2.77011 15 H 6.35001 L 7.41001 9 H 4.00511 C 3.69449 9 3.45905 8.71977 3.51259 8.41381 L 3.68759 7.41381 C 3.72946 7.17456 3.93722 7 4.18011 7 H 7.76001 L 8.39677 3.41262 C 8.43914 3.17391 8.64664 3 8.88907 3 H 9.87344 C 10.1845 3 10.4201 3.28107 10.3657 3.58738 L 9.76001 7 H 15.76 L 16.3968 3.41262 C 16.4391 3.17391 16.6466 3 16.8891 3 H 17.8734 C 18.1845 3 18.4201 3.28107 18.3657 3.58738 L 17.76 7 H 21.1649 C 21.4755 7 21.711 7.28023 21.6574 7.58619 L 21.4824 8.58619 C 21.4406 8.82544 21.2328 9 20.9899 9 H 17.41 L 16.35 15 H 19.7549 C 20.0655 15 20.301 15.2802 20.2474 15.5862 L 20.0724 16.5862 C 20.0306 16.8254 19.8228 17 19.5799 17 H 16 L 15.3632 20.5874 C 15.3209 20.8261 15.1134 21 14.8709 21 H 13.8866 C 13.5755 21 13.3399 20.7189 13.3943 20.4126 L 14 17 H 8.00001 L 7.36325 20.5874 C 7.32088 20.8261 7.11337 21 6.87094 21 H 5.88657Z M 9.41045 9 L 8.35045 15 H 14.3504 L 15.4104 9 H 9.41045 Z`,
			GUILD_VOICE: `M 11.383 3.07904 C 11.009 2.92504 10.579 3.01004 10.293 3.29604 L 6 8.00204 H 3 C 2.45 8.00204 2 8.45304 2 9.00204 V 15.002 C 2 15.552 2.45 16.002 3 16.002 H 6 L 10.293 20.71 C 10.579 20.996 11.009 21.082 11.383 20.927 C 11.757 20.772 12 20.407 12 20.002 V 4.00204 C 12 3.59904 11.757 3.23204 11.383 3.07904Z M 14 5.00195 V 7.00195 C 16.757 7.00195 19 9.24595 19 12.002 C 19 14.759 16.757 17.002 14 17.002 V 19.002 C 17.86 19.002 21 15.863 21 12.002 C 21 8.14295 17.86 5.00195 14 5.00195Z M 14 9.00195 C 15.654 9.00195 17 10.349 17 12.002 C 17 13.657 15.654 15.002 14 15.002 V 13.002 C 14.551 13.002 15 12.553 15 12.002 C 15 11.451 14.551 11.002 14 11.002 V 9.00195 Z`,
			GUILD_ANNOUNCEMENT: `M 3.9 8.26 H 2 V 15.2941 H 3.9 V 8.26 Z M 19.1 4 V 5.12659 L 4.85 8.26447 V 18.1176 C 4.85 18.5496 5.1464 18.9252 5.5701 19.0315 L 9.3701 19.9727 C 9.4461 19.9906 9.524 20 9.6 20 C 9.89545 20 10.1776 19.8635 10.36 19.6235 L 12.7065 16.5242 L 19.1 17.9304 V 19.0588 H 21 V 4 H 19.1 Z M 9.2181 17.9944 L 6.75 17.3826 V 15.2113 L 10.6706 16.0753 L 9.2181 17.9944 Z`,
			GUILD_STORE: `M 21.707 13.293l -11 -11 C 10.519 2.105 10.266 2 10 2 H 3c -0.553 0 -1 0.447 -1 1 v 7 c 0 0.266 0.105 0.519 0.293 0.707l11 11 c 0.195 0.195 0.451 0.293 0.707 0.293 s 0.512 -0.098 0.707 -0.293l7 -7 c 0.391 -0.391 0.391 -1.023 0 -1.414 z M 7 9c -1.106 0 -2 -0.896 -2 -2 0 -1.106 0.894 -2 2 -2 1.104 0 2 0.894 2 2 0 1.104 -0.896 2 -2 2 z`,
			DEFAULT: `M 11.44 0 c 4.07 0 8.07 1.87 8.07 6.35 c 0 4.13 -4.74 5.72 -5.75 7.21 c -0.76 1.11 -0.51 2.67 -2.61 2.67 c -1.37 0 -2.03 -1.11 -2.03 -2.13 c 0 -3.78 5.56 -4.64 5.56 -7.76 c 0 -1.72 -1.14 -2.73 -3.05 -2.73 c -4.07 0 -2.48 4.19 -5.56 4.19 c -1.11 0 -2.07 -0.67 -2.07 -1.94 C 4 2.76 7.56 0 11.44 0 z M 11.28 18.3 c 1.43 0 2.61 1.17 2.61 2.61 c 0 1.43 -1.18 2.61 -2.61 2.61 c -1.43 0 -2.61 -1.17 -2.61 -2.61 C 8.68 19.48 9.85 18.3 11.28 18.3 z`
		};
		
		const UserRowComponent = class UserRow extends BdApi.React.Component {
			componentDidMount() {
				if (this.props.user.fetchable) {
					this.props.user.fetchable = false;
					BDFDB.LibraryModules.UserFetchUtils.getUser(this.props.user.id).then(fetchedUser => {
						this.props.user = Object.assign({}, fetchedUser, BDFDB.LibraryModules.MemberStore.getMember(this.props.guildId, this.props.user.id) || {});
						BDFDB.ReactUtils.forceUpdate(this);
					});
				}
... (679 lines left)
