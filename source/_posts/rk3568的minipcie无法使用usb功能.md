title: rk3568的minipcie无法使用usb功能
date: '2024-08-30 14:43:12'
updated: '2024-08-30 14:43:15'
tags:
  - rk3568
categories:
  - kernel
  - rockchip
---
```bash
From a8f5220957ebf1e7b45b9093319e4a5a9da13024 Mon Sep 17 00:00:00 2001
From: Troy Mitchell <TroyMitchell988@gmail.com>
Date: Fri, 30 Aug 2024 14:11:28 +0800
Subject: [PATCH 1/1] modify rk3568-hbis-p68.dtsi: 	add usb hub regulator
 node and the controling gpio that enables the hub.

Signed-off-by: Troy Mitchell <TroyMitchell988@gmail.com>
---
 .../rk3568/kernel/dts/rk3568-hbis-p68.dtsi    | 86 +++++++++++++++++--
 1 file changed, 81 insertions(+), 5 deletions(-)

diff --git a/device/board/hihope/rk3568/kernel/dts/rk3568-hbis-p68.dtsi b/device/board/hihope/rk3568/kernel/dts/rk3568-hbis-p68.dtsi
index 7bc2f81..fa5c3d4 100644
--- a/device/board/hihope/rk3568/kernel/dts/rk3568-hbis-p68.dtsi
+++ b/device/board/hihope/rk3568/kernel/dts/rk3568-hbis-p68.dtsi
@@ -38,9 +38,9 @@
 		};
 	};
 
-	dc_5v: dc-5v {
+	dc_12v: dc-12v {
 		compatible = "regulator-fixed";
-		regulator-name = "dc_5v";
+		regulator-name = "dc_12v";
 		regulator-always-on;
 		regulator-boot-on;
 		regulator-min-microvolt = <5000000>;
@@ -54,7 +54,7 @@
 		regulator-boot-on;
 		regulator-min-microvolt = <5000000>;
 		regulator-max-microvolt = <5000000>;
-		vin-supply = <&dc_5v>;
+		vin-supply = <&dc_12v>;
 	};
 
 	vcc3v3_sys: vcc3v3-sys {
@@ -66,7 +66,17 @@
 		regulator-max-microvolt = <3300000>;
 		vin-supply = <&vcc5v0_sys>;
 	};
-
+	
+	vcc5v0_usb: vcc5v0-usb {
+		compatible = "regulator-fixed";
+		regulator-name = "vcc5v0_usb";
+		regulator-always-on;
+		regulator-boot-on;
+		regulator-min-microvolt = <5000000>;
+		regulator-max-microvolt = <5000000>;
+		vin-supply = <&dc_12v>;
+	};
+/*
 	vcc5v0_usb20_host: vcc5v0-usb20-host-regulator {
 		compatible = "regulator-fixed";
 		enable-active-high;
@@ -98,6 +108,51 @@
 		pinctrl-0 = <&vcc5v0_otg_vbus_en>;
 		regulator-name = "vcc5v0_otg_vbus";
 	};
+*/
+
+	vcc5v0_usb20_hub: vcc5v0-usb20-hub-regulator {
+		compatible = "regulator-fixed";
+		enable-active-high;
+		gpio = <&gpio3 RK_PA6 GPIO_ACTIVE_HIGH>;
+		pinctrl-names = "default";
+		pinctrl-0 = <&usb20_hub_pwr_en>;
+		regulator-name = "vcc5v0_usb20_hub";
+		regulator-always-on;
+		vin-supply = <&vcc5v0_usb>;
+	};
+
+	vcc5v0_usb30_host: vcc5v0-usb30-host-regulator {
+		compatible = "regulator-fixed";
+		enable-active-high;
+		gpio = <&gpio0 RK_PD6 GPIO_ACTIVE_HIGH>;
+		pinctrl-names = "default";
+		pinctrl-0 = <&usb30_host1_pwr_en>;
+		regulator-name = "vcc5v0_usb30_host";
+		regulator-always-on;
+		vin-supply = <&vcc5v0_usb>;
+	};
+
+	vcc5v0_usb20_host: vcc5v0-usb20-host-regulator {
+		compatible = "regulator-fixed";
+		enable-active-high;
+		gpio = <&gpio0 RK_PD5 GPIO_ACTIVE_HIGH>;
+		pinctrl-names = "default";
+		pinctrl-0 = <&usb20_host2_pwr_en>;
+		regulator-name = "vcc5v0_usb20_host2";
+		regulator-always-on;
+		vin-supply = <&vcc5v0_usb>;
+	};
+
+	vcc5v0_otg_vbus: vcc5v0-otg-vbus-regulator {
+		compatible = "regulator-fixed";
+		enable-active-high;
+		gpio = <&gpio0 RK_PD4 GPIO_ACTIVE_HIGH>;
+		pinctrl-names = "default";
+		pinctrl-0 = <&usb_otg_vbus_en>;
+		regulator-name = "vcc5v0_otg";
+		vin-supply = <&vcc5v0_usb>;
+	};
+
 
 	mini_pcie_3v3: mini-pcie-3v3-regulator {
 		compatible = "regulator-fixed";
@@ -1173,7 +1228,27 @@
 			rockchip,pins = <0 RK_PB0 RK_FUNC_GPIO &pcfg_pull_none>;
 		};
 	};
-	
+
+	usb {
+		usb20_hub_pwr_en: usb20-hub-pwr-en {
+			rockchip,pins = <3 RK_PA6 RK_FUNC_GPIO &pcfg_pull_none>;
+		};
+
+		usb30_host1_pwr_en: usb30-host1-pwr-en {
+			rockchip,pins =	<0 RK_PD6 RK_FUNC_GPIO &pcfg_pull_none>;
+		};
+
+		usb20_host2_pwr_en: usb20-host2-pwr-en {
+			rockchip,pins = <0 RK_PD5 RK_FUNC_GPIO &pcfg_pull_none>;
+		};
+
+		usb_otg_vbus_en: usb-otg-vbus-en {
+			rockchip,pins = <0 RK_PD4 RK_FUNC_GPIO &pcfg_pull_none>;
+		};
+	};
+
+
+/*	
 	usb {
 		vcc5v0_usb20_host_en: vcc5v0-usb20-host-en {
 			rockchip,pins = <0 RK_PD5 RK_FUNC_GPIO &pcfg_pull_none>;
@@ -1187,6 +1262,7 @@
 			rockchip,pins = <0 RK_PD3 RK_FUNC_GPIO &pcfg_pull_none>;
 		};
 	};
+*/
 /*
 	usb {
 		vcc5v0_usb20_host_en: vcc5v0-usb20-host-en {
-- 
2.34.1
```