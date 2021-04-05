// @license magnet:?xt=urn:btih:0b31508aeb0634b347b8270c7bee4d411b5d4109&dn=agpl-3.0.txt AGPL-3.0
var _wmVideos_ = new function () {
  'use strict';
  var inited = false;
  var prefix;

  function on(ev, f) {
    if (window.addEventListener) {
      window.addEventListener(ev, f, false);
    } else if (window.attachEvent) {
      window.attachEvent('on' + ev, f);
    }
  }

  function ajax(method, url, success) {
    var http = new XMLHttpRequest();
    http.onreadystatechange = function () {
      if (http.readyState === 4) {
        if (success) {
          success(http);
        }
      }
    };
    http.open(method, url);
    http.setRequestHeader('X-Requested-With', 'XMLHttpRequest');
    http.send();
  }

  function nodeRemoved(node) {
    if (typeof node.getRootNode == 'function') {
      return node.getRootNode() != document;
    } else {
      while (node.parentNode) {
	if (node.parentNode == document)
	  return false;
	node = node.parentNode;
      }
      return true;
    }
  }

  this.init = function (prefix_) {
    if (inited) {
      console.log("already initialized");
      return;
    }
    inited = true;
    prefix = prefix_;

    startVideoPlayerReplacement();
    //on('load', stopVideoPlayerReplacement);
  };

  var replacementInterval = 1000;
  var replacementIntervalId;

  function startVideoPlayerReplacement() {
    var fn = replaceVideoPlayersWithIAPlayer;
    fn();
    replacementIntervalId = setInterval(fn, replacementInterval);
  }

  function stopVideoPlayerReplacement() {
    console.log('stopping running replaceVideoPlayerWithIAPlayer');
    clearInterval(replacementIntervalId);
  }

  function replaceVideoPlayersWithIAPlayer() {
    var n = players.length;
    findVideoPlayers();
    if (n != players.length) {
      console.log('players=%o', players);
    }
    players.forEach((player) => {
      if (!player.target) return;
      if (nodeRemoved(player.target)) {
	// removed from DOM tree
	player.target = null;
	return;
      }
      player.discoverVideoInfo();
      if (!player.replaced) {
	player.replacePlayer();
      }
    });
  }

  class VideoPlayer {
    constructor(el, vinfo) {
      this.target = el;
      if (vinfo) {
	this.vinfo = vinfo;
      }
    }
    playerNode() {
      var embed = this.target.querySelector('embed#movie_player');
      if (embed) {
	return embed;
      }
      var html5 = this.target.querySelector('div.html5-video-player');
      if (html5) {
	return html5;
      }
      return this.target;
    }
    discoverVideoInfo() {
      if (!this.vinfo) {
	var vinfo = getVideoInfo(this.target);
	if (vinfo) {
	  this.vinfo = vinfo;
	  this.getVideoStreamInfo();
	}
      }
    }
    getVideoStreamInfo() {
      var vinfo = this.vinfo;
      var vstreamUrl = resourcekey('yt/' + vinfo.vid);
      __wm.ajax('HEAD', vstreamUrl, (http) => {
	vinfo.status = http.status;
	if (http.status != 200) {
	  console.log('%s -> %s', vstreamUrl, http.status);
	  this.displayError(
	    'Sorry, the Wayback Machine does not have this video (' +
	      vinfo.vid + ') archived (or not indexed yet).');
	  return;
	}
	var location = http.getResponseHeader('Content-Location');
	vinfo.location = location || http.responseURL || vstreamUrl;
	console.log('vinfo.location=%s', vinfo.location);

	var ctype = http.getResponseHeader('Content-Type');
	vinfo.ctype = ctype;
	if (ctype.indexOf('webm') >= 0) {
	  vinfo.ptype = 'webm';
	} else if (ctype.indexOf('flv') >= 0) {
	  vinfo.ptype = 'flv';
	} else if (ctype.indexOf('mp4') >= 0) {
	  vinfo.ptype = 'video/mp4';
	} else {
	  vinfo.ptype = 'webm';
	}
      });
    }
    replacePlayer() {
      if (this.vinfo && this.vinfo.location) {
	// change id of target to prevent youtube code further modifying
	// the node. if id is "watch-player", this is an old watch page in which
	// element loses important styling if id changes. don't change id in
	// this case.
	if (this.target.id != "watch-player") {
	  this.target.id = '--wm--' + (this.target.id || Date.now());
	}
	if (writeWatchPlayer(this)) {
	  this.replaced = true;
	}
      }
    }
    displayError(msg) {
      var e = document.createElement('div');
      e.setAttribute('style', 'position:absolute;top:0;background:#000000c0;' +
		     'z-index:9999;padding:20px;left:10%;right:10%;' +
		     'text-align:center;height:10%;color:#fff;' +
		     'border-bottom-left-radius:20px;' +
		     'border-bottom-right-radius:20px;' +
		     'border:solid 4px #ffffff;border-top:none;' +
		     'font-size:12pt;');
      e.innerHTML = msg;

      var msgParent = this.target;
      msgParent.insertBefore(e, msgParent.firstChild);
    }
  }

    // video players found on the page
  var players = [];

  function addVideoPlayer(el, vinfo) {
    if (el._wm_videoplayer) return false;

    var vo = new VideoPlayer(el, vinfo);
    el._wm_videoplayer = vo;
    // marker for development use
    el.classList.add('wm-videoplayer');
    players.push(vo);
    return true;
  }
  function collectVideoPlayers(query, vinfoFn) {
    vinfoFn = vinfoFn || function() { return null; }
    document.querySelectorAll(query).forEach((node) => {
      if (addVideoPlayer(node, vinfoFn(node))) {
	console.log('player el=%o %o', node, node._wm_videoplayer);
	// if (window.ytplayer) {
	//   window.ytplayer.config = {};
	//   window.ytplayer.load = function() {};
	// }
      }
    });
  }

  // discover new video players, create VideoPlayer object.
  function findVideoPlayers() {
    // this function shall find a stable anchor node containing youtube
    // video player, which is an embed element for the flash player, and
    // a node containing html5 video element and decorations around it.
    //console.log('%s findVideoPlayers', document.location.href);
    var embed_match = document.location.href.match(
      /\/embed\/([-_a-zA-Z0-9]{11})/);
    if (embed_match) {
      console.log('this is a embed for %s', embed_match[1]);
      // videos embedded in non-YouTube pages with IFRAME
      // get video ID from the URL
      collectVideoPlayers('#player', () => { { vid: embed_match[1] } });
    } else {
      collectVideoPlayers('div#watch-player.flash-player');
      collectVideoPlayers('div#watch7-player.flash-player');
      if (document.querySelector('ytd-app')) {
	collectVideoPlayers('ytd-app ytd-player');
      } else {
	collectVideoPlayers('#player-api');
      }
    }
  }

  var reVideoId1 = /\?video_id=([-_a-zA-Z0-9]+)/;
  var reVideoId2 = /\&video_id=([-_a-zA-Z0-9]+)/;

  var videoinfoDiscoveryMethods = [
    function (vp) {
      var video = vp.querySelector('video');
      var src = (video || vp).src;
      console.log('video.src = %s', src);
      if (src) {
	var m = src.match(reVideoId1);
	if (m) {
          return {vid: m[1]};
	}
      }
      return null;
    },
    function (vp) {
      // embed with Flash player
      if (vp.tagName != 'EMBED') {
	vp = document.querySelector('embed');
	if (!vp) {
	  return null;
	}
      }
      var flashvars = vp.getAttribute('flashvars');
      console.log('attr flashvars = %s', flashvars);
      if (flashvars) {
	var m = flashvars.match(reVideoId2);
	if (m) {
	  return {vid: m[1]};
	}
      }
      return null;
    },
    function (vp) {
      var thumbnail = vp.querySelector('div.ytp-cued-thumbnail-overlay-image');
      console.log('thumbnail = %o', thumbnail);
      if (thumbnail) {
	var bgImageStyle = window.getComputedStyle(thumbnail).backgroundImage;
	if (bgImageStyle) {
	  var m = bgImageStyle.match(/\/vi\/([-_a-zA-Z0-9]+)\/hqdefault/);
	  if (m) {
	    return {vid: m[1]};
	  }
	}
      }
      return null;
    },
    function (vp) {
      // found on recent Polymer-based rendering
      var ytd_watch = findParentTag(vp, 'YTD-WATCH-FLEXY');
      console.log('ytd-watch-flexy = %o', ytd_watch);
      if (ytd_watch) {
	var  vid = ytd_watch.getAttribute('video-id');
	// this attribute is set for age-restricted videos, and activates
	// visibility: hidden on #player-container-outer.ytd-watch-flexy
	// element (thus makes jwplayer invisible).
	if (ytd_watch.hasAttribute("player-unavailable")) {
	  ytd_watch.removeAttribute("player-unavailable");
	  var error_screen = ytd_watch.querySelector('#error-screen');
	  if (error_screen) {
	    // there will also be
	    // yt-playability-error-supported-renderers#error-screen element
	    // obscuring player element <ytd-player>.
	    error_screen.parentElement.removeChild(error_screen);
	  } else {
	    console.log('#error-screen element is not found')
	  }
	}
	if (vid) {
	  return {vid: vid};
	}
      }
      return null;
    },
    function (vp) {
      // find videoId in meta element
      var content = document.querySelector('div#content meta[itemprop="videoId"]');
      console.log('div#content meta[itemprop="vidoId"] = %o', content);
      if (content) {
	var vid = content.getAttribute('content');
	if (vid) {
          return {vid: vid};
	}
      }
      return null;
    },
    function (vp) {
      // watch page in 2012 does not have ytplayer. using ytplayer in
      // compound expression results in "undefined" error.
      if (typeof ytplayer !== 'undefined') {
	// get videoId from youtube player config.
	if (ytplayer.config && ytplayer.config.args) {
	  var vid = ytplayer.config.args.video_id;
	  if (vid) {
            return {vid: vid};
	  }
	}
      }
    }
  ];

  function getVideoInfo(vp) {
    for (var i = 0; i < videoinfoDiscoveryMethods.length; i++) {
      var vinfo = videoinfoDiscoveryMethods[i](vp);
      if (vinfo) {
	console.log('  vid=%s', vinfo.vid);
	return vinfo;
      }
    }
    return null;
  }

  function findParentTag(el, tag) {
    var parent = el.parentElement;
    while (parent && parent != document.body) {
      if (parent.tagName == tag) {
	return parent;
      }
      parent = parent.parentElement;
    }
    return null;
  }

  function removePreviousEls(els) {
    els.forEach(function(el) {
      el.remove();
    });
  }

  function is_mac_safari() {
    return (navigator.userAgent.indexOf('Safari') != -1 &&
            navigator.userAgent.indexOf('Mac') != -1 &&
            navigator.userAgent.indexOf('Chrome') == -1);
  }

  function doSetup(vo, theType, autostart, onErrFallback) {
    if (!jwplayer) {
      console.log('jwplayer is not loaded');
    }

    var player = vo.playerNode();
    if (!player) return;

    console.log('initializing jwplayer on %o (vo=%o)', player, vo);
    jwplayer.key = "sZbikYlI/xtW0U/3Tw1DOdjC1EahhtUCJF5KggVdqDY=";

    // give id to player node if it does not have one. necessary for jwplayer.
    if (!player.id) {
      player.id = '--wm--' + Date.now();
    }

    var jwpInstance = jwplayer(player.id);

    if (!jwpInstance.setup) {
      console.log('failed to initialize jwplayer');
      vo.replaced = false;
      return;
    }

    var width = player.clientWidth || "100%";
    var height = player.clientHeight || "100%";

    jwpInstance.setup({
      height: height,
      width: width,
      autostart: autostart,
      image: vo.fullImgUrl,
      primary: is_mac_safari() ? 'flash' : 'html5',
      playlist: [{
        image: vo.fullImgUrl,
        sources: [{
          'file': vo.vinfo.location,
          type: vo.vinfo.ptype
        }]
      }],
      events: {
        onError: onErrFallback
      },
      analytics: {
        enabled: false
      }
    });
  }


  const HIJACK_YOUTUBE_PLAYER = false;

  function writeWatchPlayer(vo) {
    var vinfo = vo.vinfo;
    if (!vinfo || !vinfo.location) {
      return false;
    }
    var player = vo.playerNode();
    if (!player) {
      return false;
    }

    vo.fullImgUrl = resourcekey('yt/img/' + vo.vinfo.vid);

    // Clear youtube player config data to stop it from keep sending requests
    // to vstream on the live web (and blocked by CSP). Probably this can be
    // done much earlier.
    if (window.ytplayer) {
      window.ytplayer.config = {};
      window.ytplayer.load = function() { };
    }

    if (HIJACK_YOUTUBE_PLAYER) {
      // experimental attempt to hijack ytplayer to play archived video.
      // works okay for the most part, but breaks if user tries to switch
      // video quality, for example.
      var video = player.querySelector('video.html5-main-video');
      if (video) {
	console.log('replacing src of %o with %s', video, vinfo.location);
	video.src = vinfo.location;
	return true;
      }
    }

    vo.replaced = true;
    doSetup(vo, vinfo.ptype, false);
  }

  function resourcekey(key) {
    // TODO: we may want to return a link to "closest" capture.
    return prefix + '/2oe_/http://wayback-fakeurl.archive.org/' + key;
  }
};
// @license-end
