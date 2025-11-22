# Fingerprint Reader Setup for Lenovo IdeaPad 3 14ALC8 on Fedora

This document outlines the steps to get the Elan fingerprint reader (ID `04f3:0c4b`) on a Lenovo IdeaPad 3 14ALC8 working on Fedora, particularly after encountering issues with `libfprint` and `fprintd` installations.

## 1. Ensure `fprintd` is Installed and Running

The `fprintd` service is essential for managing fingerprint readers. We will ensure it is installed and running.

### 1.1 Verify `fprintd` Installation

First, check if `fprintd` is installed:

```bash
rpm -q fprintd
```

If it reports "package fprintd is not installed", proceed to the next step. If it shows a version, it means `fprintd` is installed.

### 1.2 Install `fprintd` (if not installed)

If `fprintd` was not installed, install it using `dnf`. We'll use `--allowerasing` to handle potential conflicts with `fprintd-clients` or similar packages.

```bash
sudo dnf install fprintd --allowerasing
```

If you encounter issues where `dnf` claims "Nothing to do" despite `rpm -q` showing it's not installed, try clearing the `dnf` cache:

```bash
sudo dnf clean all
```

Then attempt the `dnf install fprintd --allowerasing` command again.

### 1.3 Verify `fprintd` Service Status

Check the status of the `fprintd` service:

```bash
systemctl status fprintd
```

If it's not `active (running)`, try starting it:

```bash
sudo systemctl start fprintd
```

Note: `fprintd.service` is typically D-Bus activated and may not have explicit `WantedBy=` or `RequiredBy=` entries for `systemctl enable`. If you see a message about "no installation config", it's usually fine as long as the service starts.

## 2. Identify Fingerprint Reader Hardware

Confirm your fingerprint reader's vendor and product ID. For the Lenovo IdeaPad 3 14ALC8, it's typically an Elan device.

```bash
lsusb
```

Look for an entry similar to `ID 04f3:0c4b Elan Microelectronics Corp. ELAN:Fingerprint`.

## 3. Install Proprietary Elan Driver

The Elan fingerprint reader often requires a proprietary driver that isn't included in official Fedora repositories.

### 3.1 Download the Driver

Download the driver from the Lenovo support website.

```bash
wget https://download.lenovo.com/pccbbs/mobiles/r1slf01w.zip
```

### 3.2 Extract the Driver File

Locate the downloaded `.zip` file. It might be in a temporary directory.

```bash
find / -name r1slf01w.zip 2>/dev/null
```

Once found, extract the contents. Note that the initial `.zip` contains another `.zip` file.

```bash
unzip <path_to_r1slf01w.zip> -d <temporary_extraction_directory>
unzip <temporary_extraction_directory>/libfprint-2-tod1-elan_0.0.8_Ubuntu22.04.zip -d <temporary_extraction_directory>
```
*(Replace `<path_to_r1slf01w.zip>` and `<temporary_extraction_directory>` with the actual paths.)*

The key file extracted will be `libfprint-2-tod1-elan.so`.

## 4. Install `libfprint-tod` from COPR Repository

The `libfprint-2-tod1` package (often part of `libfprint-tod`) is necessary for the proprietary driver. It's usually found in COPR repositories for Fedora.

### 4.1 Enable COPR Repository

Add the `manciukic/libfprint-tod-goodix` COPR repository. While this COPR mentions "goodix", it provides the `libfprint-tod` package which may be compatible.

```bash
sudo dnf copr enable manciukic/libfprint-tod-goodix
```
Confirm by typing `y` when prompted.

### 4.2 Install `libfprint-tod-goodix`

Install the package. If it conflicts with existing `libfprint` packages, use `--allowerasing`.

```bash
sudo dnf install libfprint-tod-goodix --allowerasing
```

If you previously had `libfprint` installed from official repos and `dnf` struggles to remove it, you might need to forcibly remove it first:

```bash
sudo rpm -e --nodeps libfprint
sudo dnf install libfprint-tod-goodix
```

## 5. Copy Proprietary Driver to System Directory

Create the necessary directory and copy the extracted `.so` file.

```bash
sudo mkdir -p /usr/lib/x86_64-linux-gnu/libfprint-2/tod-1/
sudo cp <temporary_extraction_directory>/libfprint-2-tod1-elan.so /usr/lib/x86_64-linux-gnu/libfprint-2/tod-1/
```
*(Replace `<temporary_extraction_directory>` with the actual path where you extracted the driver.)*

## 6. Restart Fingerprint Service

Restart `fprintd` to load the newly installed driver.

```bash
sudo systemctl restart fprintd
```

## 7. Enroll and Verify Fingerprint

Finally, enroll your fingerprint and verify it.

### 7.1 Enroll Fingerprint

```bash
fprintd-enroll $(whoami)
```

Follow the on-screen prompts, repeatedly swiping your finger.

### 7.2 Verify Fingerprint

```bash
fprintd-verify $(whoami)
```
