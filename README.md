**پارسی** | [English](README.en.md)




# راهنمای جامع نصب، راه‌اندازی و خودکارسازی Psiphon-Core به همراه پنل گرافیکی LuCI در OpenWrt 25

این پروژه یک راهنمای کاملاً بومی و عملیاتی برای کامپایل، کانفیگ و اتصال هسته لینوکسی سایفون (`psiphon-core`) به رابط کاربری گرافیکی لوسی (**LuCI JavaScript**) در سیستم‌عامل **OpenWrt 25** است. تمامی کلیدهای کنترل سرویس، فیلدهای تنظیمات (پورت‌ها، کشور، پروتکل) و بخش مانیتورینگ وضعیت آی‌پی کاملاً همگام‌سازی شده‌اند.

---

## 🛠️ ۱. آموزش کامپایل فایل باینری (روی کامپیوتر) - PowerShell

برای ساخت فایل اجرایی اختصاصی روتر خود، ابتدا مطمئن شوید زبان Go روی سیستم شما نصب است. سپس ترمینال را باز کرده و بر اساس معماری پردازنده روتر خود، دستورات زیر را اجرا کنید:

```powershell
# دریافت سورس کد رسمی هسته سایفون از مخزن گیت‌هاب
git clone https://github.com/Psiphon-Labs/psiphon-tunnel-core.git
cd psiphon-tunnel-core/ConsoleClient

# کامپایل برای روترهای ۶۴ بیتی (Aarch64 / ARM64 مانند GL.iNet MT3000 / MT2500)
$env:GOOS="linux"
$env:GOARCH="arm64"
go build -o psiphon-core main.go

# کامپایل برای روترهای ۳۲ بیتی (ARMv7 مانند Google Wifi AC-1304)
$env:GOOS="linux"
$env:GOARCH="arm"
$env:GOARM="7"
go build -o psiphon-core main.go


```

*پس از اتمام کامپایل، فایل خروجی `psiphon-core` را از طریق ابزارهایی مانند MobaXterm یا SCP به مسیر `/usr/bin/` روی روتر منتقل کنید.*

---

## 🚀 ۲. آماده‌سازی و نصب پیش‌نیازها در OpenWrt 25

در نسخه OpenWrt 25 ابزار قدیمی `opkg` حذف شده و سیستم از **`apk`** استفاده می‌کند. با اجرای دستور زیر، مجوزهای فایل باینری را صادر کرده و ابزارهای مانیتورینگ و شبکه را نصب کنید:
⚠️⚠️ بعد از اجرای دستور زیر محتوای پوشه psiphon_data.zip به آدرس /usr/bin/psiphon_data/ منتقل کنید  ⚠️⚠️

```bash
# صدور مجوز اجرا و ساخت پوشه دیتابیس سایفون
chmod +x /usr/bin/psiphon-core
mkdir -p /usr/bin/psiphon_data

# نصب پیش‌نیازهای شبکه و ابزار مانیتورینگ آی‌پی (مخصوص OpenWrt 25)
apk -U add socat curl wget-ssl coreutils-nohup


```



---

## 📁 ۳. استقرار زیرساخت و کدهای کامل پنل گرافیکی LuCI

بلوک کد زیر یک اسکریپت همه‌کاره است. آن را به طور کامل کپی کرده و در ترمینال روتر پیست کنید. این اسکریپت تمام فایل‌های ساختاری لوسی، تنظیمات UCI، کدهای جاوااسکریپت داشبورد (همراه با دکمه‌ها و فیلدهای کامل) و مجوزهای امنیتی ACL را به صورت یکجا ایجاد می‌کند:

```bash


cat << 'EOF' > /www/luci-static/resources/view/services/psiphon.js
'use strict';
'require view';
'require form';
'require fs';
'require ui';
'require uci';
'require poll';
return L.view.extend({
	router_ip: '192.168.18.1',

	flags: {
		AT: '🇦🇹', BE: '🇧🇪', CA: '🇨🇦', CH: '🇨🇭', DE: '🇩🇪', DK: '🇩🇰', 
		ES: '🇪🇸', FI: '🇫🇮', FR: '🇫🇷', GB: '🇬🇧', IT: '🇮🇹', JP: '🇯🇵', 
		NL: '🇳🇱', NO: '🇳🇴', PL: '🇵🇱', SE: '🇸🇪', SG: '🇸🇬', US: '🇺🇸'
	},

	load: function() {
		return L.uci.load('network').then(L.bind(function() {
			var ip = L.uci.get('network', 'lan', 'ipaddr');
			if (ip) {
				this.router_ip = ip;
			}
			
			if (!document.getElementById('noto-emoji-font')) {
				var link = document.createElement('link');
				link.id = 'noto-emoji-font';
				link.rel = 'stylesheet';
				link.href = 'https://fonts.googleapis.com/css2?family=Noto+Color+Emoji&display=swap';
				document.head.appendChild(link);
				
				var style = document.createElement('style');
				style.innerHTML = `
					.win-flag { font-family: "Noto Color Emoji", "Segoe UI Emoji", "Apple Color Emoji", sans-serif !important; }
					
					/* QR Code Hover Styles with matching GL.iNet Blue/Cyan Theme */
					.qr-trigger-wrapper {
						display: inline-flex;
						align-items: center;
						gap: 6px;
						position: relative;
						margin-left: 8px;
					}
					.qr-badge {
						background: #00b4d8;
						color: #ffffff;
						font-size: 10px;
						font-weight: bold;
						padding: 3px 8px;
						border-radius: 4px;
						cursor: pointer;
						transition: all 0.25s ease-in-out;
						box-shadow: 0 2px 4px rgba(0,0,0,0.2);
						border: 1px solid rgba(255,255,255,0.1);
					}
					.qr-badge:hover {
						background: #0096c7;
						transform: scale(1.05);
						box-shadow: 0 0 8px rgba(0, 180, 216, 0.4);
					}
					.qr-popup-window {
						visibility: hidden;
						opacity: 0;
						position: absolute;
						bottom: 135%;
						left: 50%;
						transform: translateX(-50%) translateY(10px);
						background: #14171f;
						border: 1px solid #00b4d8;
						border-radius: 12px;
						padding: 12px;
						box-shadow: 0 10px 30px rgba(0,0,0,0.8), 0 0 15px rgba(0, 180, 216, 0.15);
						z-index: 9999;
						width: 180px;
						text-align: center;
						transition: opacity 0.3s cubic-bezier(0.4, 0, 0.2, 1), transform 0.3s cubic-bezier(0.4, 0, 0.2, 1), visibility 0.3s;
						pointer-events: none;
					}
					.qr-trigger-wrapper:hover .qr-popup-window {
						visibility: visible;
						opacity: 1;
						transform: translateX(-50%) translateY(0);
						pointer-events: auto;
					}
					.qr-popup-window img {
						width: 150px;
						height: 150px;
						border-radius: 8px;
						background: #fff;
						padding: 8px;
						display: block;
						margin: 0 auto 10px auto;
						box-shadow: inset 0 0 5px rgba(0,0,0,0.2);
					}
					.qr-popup-window span {
						font-size: 11px;
						color: #00b4d8;
						word-break: break-all;
						font-family: monospace;
						display: block;
						background: rgba(0, 180, 216, 0.08);
						padding: 4px 6px;
						border-radius: 4px;
						border: 1px solid rgba(0, 180, 216, 0.15);
					}
				`;
				document.head.appendChild(style);
			}
		}, this));
	},

	render: function() {
		const self = this;
		const logoContainer = E('div', { 
			'id': 'psiphon-dynamic-logo',
			'style': 'width: 64px; height: 64px; flex-shrink: 0; display: flex; align-items: center; justify-content: center;' 
		});
		function getLogoSvg(primaryColor, secondaryColor) {
			return `<svg width="64" height="64" viewBox="0 0 48.00 48.00" xmlns="http://www.w3.org/2000/svg" fill="#000000" stroke="#000000" stroke-width="0">
				<path d="M 12 0 H 36 A 12 12 0 0 1 48 12 V 36 A 12 12 0 0 1 36 48 H 12 A 12 12 0 0 1 0 36 V 12 A 12 12 0 0 1 12 0 Z" fill="${primaryColor}" stroke-width="0"/>
				<path d="M 20.593 32.889 L 34.273 32.824 L 37.97 29.341 L 40.491 9.289 L 37.302 5.922 L 10.97 5.601 C 9.476 6.667 8.285 8.104 7.514 9.77 L 17.57 9.868 L 13.607 40.876 C 15.408 41.841 17.408 42.377 19.45 42.444 L 
20.593 32.889 Z" style="stroke-width: 0.192; fill: rgb(255, 255, 255); paint-order: stroke; stroke: none;"/>
				<path d="M 23.797 11.765 L 21.624 26.41 L 31.601 26.303 L 33.739 11.765 L 23.797 11.765 Z" style="stroke-width: 0.192; fill-rule: nonzero; paint-order: stroke markers; fill: ${secondaryColor}; stroke: none;"/>
			</svg>`;
		}

		logoContainer.innerHTML = getLogoSvg('#fb510c', '#f45825');

		const titleHtml = E('div', { 'style': 'display: flex; align-items: center; gap: 16px; margin-bottom: 20px; padding: 10px 0;' }, [
			logoContainer,
			E('div', { 'style': 'display: flex; flex-direction: column; justify-content: center;' }, [
				E('h2', { 'style': 'margin: 0; font-weight: bold; color: #fff; font-size: 26px; line-height: 1.2;' }, _('Psiphon VPN Configuration')),
				E('span', { 'style': 'font-size: 12px; color: #aaa; display: block; margin-top: 5px;' }, _('Unified Single-Page Control Panel for Psiphon Tunnel Core.'))
			])
		]);
		const m = new L.form.Map('psiphon', titleHtml);
		const s = m.section(L.form.NamedSection, 'config', 'psiphon', _('Service Status & Configuration'));
		s.addremove = false;
		let o = s.option(L.form.DummyValue, '_ip_box');
		o.rawhtml = true;
		o.render = function() {
			return E('div', { 'class': 'cbi-value', 'style': 'margin-bottom: 20px;' }, [
				E('div', { 'style': 'display: flex; flex-flow: row wrap; align-items: center; justify-content: space-between; background: #1a1c20; padding: 14px 20px; border-radius: 8px; border: 1px solid #333; box-shadow: 0 4px 6px rgba(0,0,0,0.3); width: 100%; gap: 15px;' }, [
					
					// Real IP
					E('div', { 'style': 'display: flex; align-items: center; gap: 8px; flex: 1 1 0%; min-width: 220px;' }, [
						E('span', { 'style': 'color: #88a; font-size: 13px; text-transform: uppercase; font-weight: bold; width: 65px; flex-shrink: 0;' }, _('Real IP')),
						E('span', { 'id': 'real_ip_display', 'style': 'font-size: 14px; color: #ccc; font-family: monospace; display: flex; align-items: center; gap: 6px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap;' }, _('Checking'))
					]),

					E('div', { 'style': 'width: 1px; height: 24px; background-color: #444; display: inline-block;' }, ''),

					// Psiphon IP
					E('div', { 'style': 'display: flex; align-items: center; gap: 8px; flex: 1 1 0%; min-width: 220px;' }, [
						E('span', { 'style': 'color: #88a; font-size: 13px; text-transform: uppercase; font-weight: bold; width: 85px; flex-shrink: 0;' }, _('Psiphon IP')),
						E('span', { 'id': 'vpn_ip_display', 'style': 'font-size: 14px; color: #00ff66; font-family: monospace; display: flex; align-items: center; gap: 6px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap;' }, _('Checking'))
					]),

					// Refresh Button
					E('div', { 'style': 'flex-shrink: 0;' }, [
						E('button', {
							'class': 'btn cbi-button cbi-button-apply',
							'style': 'padding: 6px 15px; font-weight: bold; white-space: nowrap;',
							'click': function(ev) {
								ev.preventDefault();
								refreshIPs();
							}
						}, _('Refresh'))
					])
				])
			]);
		};

		function updateDisplayElement(el, data, fallbackText, successColor) {
			if (el == null) return;
			if (data && data.query) {
				const cc = (data.countryCode || data.country_code || data.country || '').toUpperCase();
				const countryName = data.countryName || data.country || '';
				el.innerHTML = '';
				
				const flagEmoji = self.flags[cc] || '';
				if (flagEmoji) {
					el.appendChild(E('span', { 'class': 'win-flag', 'style': 'font-size: 18px; line-height: 1; vertical-align: middle;' }, flagEmoji));
				}
				
				el.appendChild(E('b', { 'style': 'font-size: 15px; margin-left: 6px; vertical-align: middle; color: ' + (successColor || '#ccc') }, ' ' + data.query));
				if (countryName) {
					el.appendChild(E('span', { 'style': 'color: #888; font-size: 12px; margin-left: 5px; white-space: nowrap; vertical-align: middle;' }, '(' + countryName + ')'));
				}
			} else {
				el.textContent = fallbackText;
			}
		}

		function refreshIPs() {
			const elReal = document.getElementById('real_ip_display');
			const elVpn = document.getElementById('vpn_ip_display');
			const elLogo = document.getElementById('psiphon-dynamic-logo');
			if (elReal) elReal.textContent = _('Checking');
			if (elVpn) elVpn.textContent = _('Checking');
			fetch('https://ipinfo.io/json', { method: 'GET', mode: 'cors' })
				.then(function(res) { return res.json(); })
				.then(function(data) {
					const formatted = {
						status: 'success',
						query: data.ip,
						countryCode: data.country,
						country: data.country
					};
					updateDisplayElement(elReal, formatted, _('Failed'));
				})
				.catch(function() {
					fetch('https://ip-api.com/json/', { method: 'GET', mode: 'cors' })
						.then(function(res) { return res.json(); })
						.then(function(data) {
							updateDisplayElement(elReal, data, _('Failed'));
						})
						.catch(function() {
							L.fs.exec('/bin/sh', ['-c', 'curl -sL -m 4 https://ipinfo.io/json || curl -sL -m 4 http://api.com/json/ || true']).then(function(res) {
								try {
									if (res.stdout && res.stdout.trim() !== '') {
										const parsed = JSON.parse(res.stdout);
										const formatted = {
											status: 'success',
											query: parsed.ip || parsed.query,
											countryCode: parsed.country || parsed.countryCode,
											country: parsed.country || parsed.countryName
										};
										updateDisplayElement(elReal, formatted, _('Network Error'));
									} else {
										if (elReal) elReal.textContent = _('Network Error');
									}
								} catch(e) {
									if (elReal) elReal.textContent = _('Failed');
								}
							});
						});
				});

			const current_lan_ip = self.router_ip || '192.168.18.1';
			const cmdVpn = 'curl -sL -m 5 -x http://' + current_lan_ip + ':10809 http://ip-api.com/json/ || curl -sL -m 5 --socks5 ' + current_lan_ip + ':10808 http://ip-api.com/json/ || curl -sL -m 5 -x http://' + current_lan_ip + ':10809 https://ipinfo.io/json || true';
			L.fs.exec('/bin/sh', ['-c', cmdVpn]).then(function(res) {
				try {
					if (res.stdout && res.stdout.trim() !== '') {
						const parsed = JSON.parse(res.stdout);
						const formatted = {
							status: 'success',
							query: parsed.query || parsed.ip,
							countryCode: parsed.countryCode || parsed.country,
							country: parsed.country || parsed.countryName
						};
						updateDisplayElement(elVpn, formatted, _('Disconnected'), '#00ff66');
						
						if (elLogo) {
							elLogo.innerHTML = getLogoSvg('#00e873', '#00e873');
						}
					} else {
						if (elVpn) elVpn.textContent = _('Disconnected');
						if (elLogo) {
							elLogo.innerHTML = getLogoSvg('#fb510c', '#f45825');
						}
					}
				} catch(e) { 
					if (elVpn) elVpn.textContent = _('Disconnected'); 
					if (elLogo) {
						elLogo.innerHTML = getLogoSvg('#fb510c', '#f45825');
					}
				}
			}).catch(function() {
				if (elVpn) elVpn.textContent = _('Disconnected');
				if (elLogo) {
					elLogo.innerHTML = getLogoSvg('#fb510c', '#f45825');
				}
			});
		}
		
		o = s.option(L.form.DummyValue, '_control_buttons');
		o.rawhtml = true;
		o.render = function() {
			return E('div', { 'class': 'cbi-value' }, [
				E('label', { 'class': 'cbi-value-title' }, _('Service Control')),
				E('div', { 'class': 'cbi-value-field' }, [
					E('button', { 
						'class': 'btn cbi-button cbi-button-remove', 
						'style': 'margin-right: 10px;', 
						'click': function(ev) { 
							ev.preventDefault(); 
							ui.showModal(_('Stopping service'), [ E('p', { 'class': 'spinning' }, _('Please wait')) ]);
							L.fs.exec('/etc/init.d/psiphon', ['stop']).then(function() { 
								L.fs.exec('/bin/sh', ['-c', 'killall -9 psiphon-core socat 2>/dev/null || true']).then(function() {
									ui.hideModal();
									ui.addNotification(null, E('p', _('Psiphon Service Stopped successfully')), 'info');
									refreshIPs();
								});
							}).catch(function(err) { 
								ui.hideModal();
								ui.addNotification(null, E('p', _('Stop Error ') + err), 'danger');
							}); 
						}
					}, _('Stop')),
					E('button', { 
						'class': 'btn cbi-button cbi-button-apply', 
						'click': function(ev) { 
							ev.preventDefault();
							ui.showModal(_('Starting service'), [ E('p', { 'class': 'spinning' }, _('Initializing Psiphon Core')) ]);
							
							// 
							L.fs.exec('/etc/init.d/psiphon', ['start']).then(function() {
								// 
							}).catch(function(err) {
								console.error('Start error:', err);
							});
							
							setTimeout(function() {
								ui.hideModal();
								ui.addNotification(null, E('p', _('Psiphon Service started in background. Please wait a moment for the connection.')), 'info');
								setTimeout(refreshIPs, 5000);
							}, 1000);
						}
					}, _('Start'))
				])
			]);
		};

		o = s.option(L.form.Flag, 'enabled', _('Enable'));
		o.rmempty = false;

		o = s.option(L.form.ListValue, 'country', _('Region'));
		o.rmempty = true;
		o.optional = true;

		o.value('', '⚡ ' + _('Best Performance'));

		const countries = [
			{ code: 'AT', name: 'Austria' },
			{ code: 'BE', name: 'Belgium' },
			{ code: 'CA', name: 'Canada' },
			{ code: 'CH', name: 'Switzerland' },
			{ code: 'DE', name: 'Germany' },
			{ code: 'DK', name: 'Denmark' },
			{ code: 'ES', name: 'Spain' },
			{ code: 'FI', name: 'Finland' },
			{ code: 'FR', name: 'France' },
			{ code: 'GB', name: 'United Kingdom' },
			{ code: 'IT', name: 'Italy' },
			{ code: 'JP', name: 'Japan' },
			{ code: 'NL', name: 'Netherlands' },
			{ code: 'NO', name: 'Norway' },
			{ code: 'PL', name: 'Poland' },
			{ code: 'SE', name: 'Sweden' },
			{ code: 'SG', name: 'Singapore' },
			{ code: 'US', name: 'United States' }
		];

		countries.forEach(function(c) {
			const flag = self.flags[c.code] || '';
			o.value(c.code, flag + ' [' + c.code + '] ' + c.name);
		});
		o.default = '';

		o.write = function(section_id, value) {
			return L.form.ListValue.prototype.write.call(this, section_id, value);
		};

		o = s.option(L.form.ListValue, 'transport', _('Transport Mode'));
		o.value('STANDARD', _('Standard'));
		o.value('QUIC', _('QUIC'));
		o.value('SSH', _('SSH'));
		o.default = 'STANDARD';

		// Helper to generate V2Ray Hover QR HTML (Using matching Cyan / Navy blue theme)
		function buildQrMarkup(protocol, ip, port, remarks) {
			const rawConfig = protocol + '://' + ip + ':' + port + '#' + remarks;
			const qrApiUrl = 'https://api.qrserver.com/v1/create-qr-code/?size=150x150&data=' + encodeURIComponent(rawConfig);
			return `<div class="qr-trigger-wrapper">
				<span class="qr-badge">QR CONFIG</span>
				<div class="qr-popup-window">
					<img src="${qrApiUrl}" alt="QR Code" />
					<span>${rawConfig}</span>
				</div>
			</div>`;
		}

		const socksOpt = s.option(L.form.Value, 'socks_port', _('SOCKS Port'));
		socksOpt.datatype = 'port';
		socksOpt.default = '10808';
		socksOpt.rawhtml = true;
		socksOpt.description = _('Connect clients to ') + '<b>' + self.router_ip + ':10808</b>' + 
			buildQrMarkup('socks', self.router_ip, '10808', 'Psiphon-SOCKS');
		socksOpt.write = function(section_id, value) {
			const targetPort = value || '10808';
			this.description = _('Connect clients to ') + '<b>' + self.router_ip + ':' + targetPort + '</b>' +
				buildQrMarkup('socks', self.router_ip, targetPort, 'Psiphon-SOCKS');
			return L.form.Value.prototype.write.call(this, section_id, value);
		};

		const httpOpt = s.option(L.form.Value, 'http_port', _('HTTP Port'));
		httpOpt.datatype = 'port';
		httpOpt.default = '10809';
		httpOpt.rawhtml = true;

		httpOpt.description = _('Connect clients to ') + '<b>' + self.router_ip + ':10809</b>' +
			buildQrMarkup('http', self.router_ip, '10809', 'Psiphon-HTTP');
		httpOpt.write = function(section_id, value) {
			const targetPort = value || '10809';
			this.description = _('Connect clients to ') + '<b>' + self.router_ip + ':' + targetPort + '</b>' +
				buildQrMarkup('http', self.router_ip, targetPort, 'Psiphon-HTTP');
			return L.form.Value.prototype.write.call(this, section_id, value);
		};

		o = s.option(L.form.Flag, 'beast_mode', _('Beast Mode'), _('Aggressive connection tries all protocols on every server'));
		o.rmempty = true;
		o.default = '0';

		o = s.option(L.form.Value, 'cdn_edge_ips', _('CDN edge IPs'), _('Optional CDN edge IPs separated by commas spaces or new lines'));
		o.rmempty = true;

		o = s.option(L.form.Value, 'cdn_sni', _('CDN SNI hostname'), _('Optional SNI hostname for CDN edge IPs leave blank to use the built-in behavior'));
		o.rmempty = true;

		o = s.option(L.form.Button, '_clear_log', _('Log Actions'));
		o.inputtitle = _('Clear Log Screen');
		o.inputstyle = 'reset';
		o.onclick = function(ev) {
			const box = document.getElementById('psiphon_live_log');
			if (box) box.value = _('Log monitor cleared');
			return L.fs.exec('/bin/sh', ['-c', '> /tmp/psiphon.log || true']);
		};

		o = s.option(L.form.DummyValue, '_console_and_log');
		o.rawhtml = true;
		o.render = function() {
			return E('div', {}, [
				E('div', { 'class': 'cbi-value' }, [
					E('label', { 'class': 'cbi-value-title' }, _('Logs')),
					E('div', { 'class': 'cbi-value-field', 'style': 'width: 100%; box-sizing: border-box;' }, [
						E('textarea', { 
							'id': 'psiphon_live_log', 
							'style': 'width: 100%; height: 260px; font-family: monospace; font-size: 12px; background: #111; color: #00ff66; padding: 10px; border-radius: 4px; border: 1px solid #222; resize: none; line-height: 1.6; box-sizing: border-box; margin-bottom: 15px;', 
							'readonly': 'readonly' 
						}, _('Waiting for log stream'))
					])
				]),
				
				E('div', { 'class': 'cbi-value' }, [
					E('label', { 'class': 'cbi-value-title' }, _('Terminal')),
					E('div', { 'class': 'cbi-value-field', 'style': 'width: 100%; box-sizing: border-box;' }, [
						E('textarea', { 
							'id': 'cmd_input', 
							'placeholder': 'Enter shell command here\nExample ps -w | grep psiphon', 
							'style': 'width: 100%; height: 90px; padding: 10px; margin-bottom: 10px; background: #1a1c20; color: #fff; border: 1px solid #444; font-family: monospace; resize: none; box-sizing: border-box; border-radius: 4px;' 
						}),
						E('div', { 'style': 'text-align: right;' }, [
							E('button', { 
								'class': 'btn cbi-button cbi-button-apply',
								'click': function(ev) {
									ev.preventDefault();
									const cmdInput = document.getElementById('cmd_input');
									const cmd = cmdInput ? cmdInput.value.trim() : '';
									if (cmd === '') return;

									const logArea = document.getElementById('psiphon_live_log');
									if (logArea) {
										logArea.value += '\n$ ' + cmd + '\n';
										logArea.scrollTop = logArea.scrollHeight;
									}
									
									L.fs.exec('/bin/sh', ['-c', cmd]).then(function(res) {
										if (logArea) {
											logArea.value += (res.stdout || '') + (res.stderr || '');
											logArea.scrollTop = logArea.scrollHeight;
										}
										if (cmdInput) cmdInput.value = '';
									});
								}
							}, _('Run Command'))
						])
					])
				])
			]);
		};

		function fetchLog() {
			const logArea = document.getElementById('psiphon_live_log');
			if (logArea == null) return;
			L.resolveDefault(L.fs.read('/tmp/psiphon.log'), '').then(function(res) {
				if (res && res.trim() !== '') {
					const lines = res.split('\n');
					const filteredLog = [];

					for (let i = 0; i < lines.length; i++) {
						const line = lines[i].trim();
						if (line === '') continue;

						try {
							const logObj = JSON.parse(line);
							const time = logObj.timestamp ? logObj.timestamp.substring(11, 19) : '';
							
							if (logObj.noticeType === 'ConnectedServerRegion') {
								filteredLog.push('[' + time + '] Connected to server region ' + logObj.data.serverRegion);
							} else if (logObj.noticeType === 'Tunnels') {
								filteredLog.push('[' + time + '] Tunnels Count ' + logObj.data.count);
							} else if (logObj.noticeType === 'TrafficRateLimits') {
								const down = (logObj.data.downstreamBytesPerSecond / 1024).toFixed(1);
								const up = (logObj.data.upstreamBytesPerSecond / 1024).toFixed(1);
								filteredLog.push('[' + time + '] Speed Down ' + down + ' KB/s Up ' + up + ' KB/s');
							} else if (logObj.noticeType === 'ClientRegion') {
								filteredLog.push('[' + time + '] Current Internet Location ' + logObj.data.region);
							} else if (logObj.noticeType === 'SkipServerEntry') {
								filteredLog.push('[' + time + '] Skipping Blocked Server ' + (logObj.data.reason || 'Timeout'));
							}
						} catch (e) {
							if (line.includes(' Listening') === false && line.includes('Parameters') === false && line.includes('AvailableEgress') === false && line.includes('ActiveAuthorizationIDs') === false) {
								filteredLog.push(line);
							}
						}
					}
					logArea.value = filteredLog.length > 0 ? filteredLog.join('\n') : _('Standby');
					logArea.scrollTop = logArea.scrollHeight;
				} else {
					logArea.value = _('Service stopped or log empty');
				}
			});
		}

		setTimeout(function() {
			var dropdowns = document.querySelectorAll('.cbi-input-select, select, option, .control-group');
			for (var i = 0; i < dropdowns.length; i++) {
				dropdowns[i].classList.add('win-flag');
			}
		}, 500);
		L.Poll.add(function() {
			return Promise.all([
				fetchLog()
			]);
		}, 3);

		setTimeout(function() {
			refreshIPs();
			fetchLog();
		}, 1000);

		return m.render();
	}
});
EOF

rm -rf /tmp/luci-indexcache /tmp/luci-modulecache && /etc/init.d/rpcd restart

echo "Successfully updated and fixed the start button! Please press Ctrl + F5."

ok



```

## ⚙️ ۴. ایجاد اسکریپت سرویس هوشمند سیستم (`/etc/init.d/psiphon`)

این اسکریپت مغز متفکر اتصال بک‌اند است. هنگام استارت، تمامی فیلدهای تکمیل‌شده در پنل لوسی (مانند کشور انتخابی، پورت‌ها و نوع پروتکل) را دریافت کرده، فایل کانفیگ اصلی سایفون را در لحظه بازنویسی می‌کند و پروسه هدایت اتصالات به پورت‌های محلی شبکه LAN روتر را انجام می‌دهد:

```bash

cat << 'EOF' > /etc/init.d/psiphon
#!/bin/sh /etc/rc.common

START=99
USE_PROCD=1

start_service() {
    # ۱. خواندن پیکربندی‌های ذخیره شده از uci لوسی
    config_load psiphon
    local enabled country transport socks_port http_port
    
    config_get_bool enabled config enabled 0
    config_get country config country 'ALL'
    config_get transport config transport 'STANDARD'
    config_get socks_port config socks_port '10808'
    config_get http_port config http_port '10809'
    
    # اگر تیک گزینه Enable خاموش باشد، اجرای سرویس متوقف می‌شود
    if [ "$enabled" -eq 0 ]; then
        echo "Psiphon is disabled in LuCI configuration. Skipping start." > /tmp/psiphon.log
        return 0
    fi

    # ۲. ساخت و بروزرسانی پویا و آنی فایل کانفیگ JSON سایفون بر اساس متغیرهای انتخابی لوسی
    cat << JSON > /usr/bin/psiphon.config
{
    "SocksProxyPort": 1080,
    "HttpProxyPort": 1081,
    "DataRootDirectory": "/usr/bin/psiphon_data",
    "EgressRegion": "$country",
    "TransportProtocols": ["$transport"],
    "PropagationChannelId": "0000000000000000",
    "SponsorId": "0000000000000000",
    "ServerEntrySignaturePublicKey": "sHuUVTWaRyh5pZwy4UguSgkwmBe0EHtJJkoF5WrxmvA=",
    "UseIndistinguishableTLS": true
}
JSON

    # ۳. راه‌اندازی و مدیریت پروسه هوشمند هسته با سیستم پروکد
    procd_open_instance "psiphon"
    procd_set_param command /bin/sh -c "
        killall -9 psiphon-core socat 2>/dev/null
        
        # ایجاد لاگ تمیز و اجرای هسته اصلی سایفون
        echo '' > /tmp/psiphon.log
        /usr/bin/psiphon-core -config /usr/bin/psiphon.config >> /tmp/psiphon.log 2>&1 &
        PSIPHON_PID=\$!
        
        # انتظار منطقی و هوشمند برای بالا آمدن پورت‌ها و نوشتن لاگ
        echo 'Waiting for Psiphon to establish tunnels...' >> /tmp/psiphon.log
        
        local timeout=15
        local CORE_SOCKS=''
        local CORE_HTTP=''
        
        while [ \$timeout -gt 0 ]; do
            sleep 1
            CORE_SOCKS=\$(grep 'ListeningSocksProxyPort' /tmp/psiphon.log | grep -o '\"port\":[0-9]*' | cut -d':' -f2 | head -n 1)
            CORE_HTTP=\$(grep 'ListeningHttpProxyPort' /tmp/psiphon.log | grep -o '\"port\":[0-9]*' | cut -d':' -f2 | head -n 1)
            
            if [ ! -z \"\$CORE_SOCKS\" ] && [ ! -z \"\$CORE_HTTP\" ]; then
                break
            fi
            timeout=\$((timeout - 1))
        done
        
        # واکشی آی‌پی محلی فعلی روتر (LAN IP) برای پل زدن سراسری اتصالات
        ROUTER_IP=\$(ubus call network.interface.lan status | jsonfilter -e '@[\"ipv4-address\"][0].address')
        [ -z \"\$ROUTER_IP\" ] && ROUTER_IP='192.168.18.1'
        
        # اتصال پورت‌های لوکال هاست سایفون به پورت‌های ثابت تعریف شده در لوسی بر روی کل شبکه LAN
        if [ ! -z \"\$CORE_SOCKS\" ] && [ ! -z \"\$CORE_HTTP\" ]; then
            socat TCP-LISTEN:$socks_port,fork,bind=\$ROUTER_IP TCP:127.0.0.1:\$CORE_SOCKS &
            socat TCP-LISTEN:$http_port,fork,bind=\$ROUTER_IP TCP:127.0.0.1:\$CORE_HTTP &
            echo \"Successfully bridged LAN $socks_port->Core \$CORE_SOCKS and LAN $http_port->Core \$CORE_HTTP on IP \$ROUTER_IP\" >> /tmp/psiphon.log
        else
            echo 'Failed to detect Psiphon dynamic ports!' >> /tmp/psiphon.log
        fi
        
        wait \$PSIPHON_PID
    "
    procd_set_param respawn
    procd_close_instance
}

stop_service() {
    killall -9 psiphon-core socat 2>/dev/null || true
    echo "Psiphon core and socat tunnels stopped safely." > /tmp/psiphon.log
}
EOF

# اعطای دسترسی‌های اجرایی سرویس سیستم
chmod +x /etc/init.d/psiphon
/etc/init.d/psiphon enable


```

---

## 🔄 ۵. بارگذاری نهایی و اعمال تغییرات پنل گرافیکی

برای پاک کردن سیستم کش قدیمی رابط کاربری لوسی و بارگذاری کامل اسکریپت‌های امنیتی سیستم RPC، دستورات زیر را اجرا کنید:

```bash
chmod 644 /usr/share/rpcd/acl.d/luci-app-psiphon.json
chmod 644 /usr/share/luci/menu.d/luci-app-psiphon.json

# راه‌اندازی مجدد سرویس سیستم تبادل داده و وب‌سرور لوسی
/etc/init.d/rpcd restart
/etc/init.d/uhttpd restart


```

*حالا مرورگر خود را با کلیدهای ترکیبی `Ctrl + F5` در سیستم ریفرش کنید تا منوی سرویس با عملکرد کامل کلیدها بالا بیاید. (توصیه می‌شود یک بار از پنل Log Out کرده و مجدد وارد شوید).*

---

## 🛡️ ۶. تنظیمات فایروال (مبتنی بر Nftables در OpenWrt 25)

برای اینکه پورت‌های جدید پروکسی روتر اجازه تبادل اطلاعات با سایر دستگاه‌های متصل به وای‌فای یا شبکه LAN را داشته باشند، پورت‌ها را در لایه فایروال محلی باز کنید:

```bash
# باز کردن پورت‌های ورودی فایروال شبکه برای اتصالات کلاینت‌ها
nft add rule inet fw4 input iifname "br-lan" tcp dport 10808-10809 accept 2>/dev/null || true
/etc/init.d/firewall reload


```

اگر می‌خواهید بعد از ریبوت هم قوانین فایروال پاک نشود دستور زیر را بزنید
```bash

# اضافه کردن قانون فایروال با ساختار کاملاً منظم و ایجاد خط جدید
# ۱. پاک کردن تمام خطوط مربوط به قانون سایفون (پاکسازی فایل فایروال)
sed -i '/Allow-Psiphon-Proxy/d' /etc/config/firewall
sed -i '/Allow-Psiphon/d' /etc/config/firewall

# ۲. اضافه کردن هوشمند و استاندارد قانون با فاصله‌های منظم (با استفاده از printf برای ایجاد دقیق Tab)
printf "\nconfig rule\n\toption name 'Allow-Psiphon-Proxy'\n\toption src 'lan'\n\toption dest_port '10808-10809'\n\toption proto 'tcp'\n\toption target 'ACCEPT'\n" >> /etc/config/firewall

# ۳. اعمال تغییرات در فایروال سیستم
/etc/init.d/firewall restart

```


---

## 🩺 ۷. تست و عیب‌یابی شبکه از طریق ترمینال

پس از زدن دکمه **Start Service** در پنل لوسی، با دستورات زیر می‌توانید عملکرد فیلترشکن را به طور مستقیم در لایه شبکه روتر ارزیابی کنید:

* **تست عبور موفق ترافیک از پورت HTTP Proxy:**

```bash
curl -x http://127.0.0.1:10809 https://api.ipify.org


```

* **تست عبور موفق ترافیک از پورت SOCKS5 Proxy:**

```bash
curl --socks5-hostname 127.0.0.1:10808 https://api.ipify.org


```

---

*تست اجرای و متوقف کردن با دستور*

* **روشن کردن تانل سایفون:**

```bash
/etc/init.d/psiphon start

```

* **خاموش کردن کامل سیستم:**

```bash
/etc/init.d/psiphon stop

```

* **فعال‌سازی اجرای خودکار پس از روشن شدن روتر:**

```bash
/etc/init.d/psiphon enable

```

## 🗑️ ۸. حذف کامل و بی‌بازگشت سایفون از سیستم (Uninstall)

اگر به هر دلیلی تمایل داشتید تمامی تنظیمات، فایل‌های باینری، دیتابیس‌ها و منوهای پنل لوسی سایفون را بدون به جا ماندن هیچ ردپایی حذف کنید، اسکریپت یکپارچه زیر را در ترمینال روتر اجرا کنید:

```bash
# ۱. متوقف کردن پردازش‌ها و غیرفعال‌سازی سرویس سیستم
/etc/init.d/psiphon disable 2>/dev/null
/etc/init.d/psiphon stop 2>/dev/null
killall -9 psiphon-core socat 2>/dev/null

# ۲. پاکسازی کامل فایل‌های سیستمی و پنل گرافیکی لوسی
rm -f /usr/bin/psiphon-core
rm -rf /usr/bin/psiphon_data
rm -f /usr/bin/psiphon.config
rm -f /etc/init.d/psiphon
rm -f /etc/config/psiphon
rm -f /www/luci-static/resources/view/services/psiphon.js
rm -f /usr/share/luci/menu.d/luci-app-psiphon.json
rm -f /usr/share/rpcd/acl.d/luci-app-psiphon.json
rm -f /tmp/psiphon.log

# ۳. راه‌اندازی مجدد بخش دسترسی‌ها و پاک کردن کامل سیستم کش لوسی
/etc/init.d/rpcd restart
/etc/init.d/uhttpd restart
rm -f /tmp/luci-indexcache*
rm -rf /tmp/luci-modulecache/*
rm -rf /tmp/luci-sessions/*

echo "Psiphon app and all associated LuCI components successfully uninstalled."


```

## محیط Luci برای سایفون 

<img width="2851" height="2909" alt="LUCI for Psiphon" src="https://github.com/user-attachments/assets/243e98af-7b56-406f-a521-87eeab589462" />










