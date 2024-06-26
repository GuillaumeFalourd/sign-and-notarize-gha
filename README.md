# sign-and-notarize-gha

Repository explaining step by step how to sign and notarize packages with Apple using Github Actions :apple: :octocat:

## Premisses

- An [Apple Developer Id account](https://developer.apple.com/) (email + password)
- A Developer Id **Installer** certificate.
- A Developer Id **Application** certificate.
- A repository on Github Action with the code you want to build and distribute through a package.

## Explanation

**First, it's worth highlighting the reason why 2 certificates are needed.**

- The `Developer Id Installer` certificate is used to sign your application/software **package** for distribution **outside the Mac Apple Store**.

- The `Developer Id Application` certificate is used to sign your application/software **files** (used to generate the package).

Basically, if you just sign the installation **package**, without signing the code files inside it, Apple's notarization process will refuse your package. In other words, in order to get Apple's notarization for your installation package, you need to sign an `INSIDE OUT` signature, meaning you need to sign all the files used in your package (such as binaries, libraries, scripts…).

_In short: The Developer Id Application will be used to sign the code, and the Developer Id Installer will be used to sign the package. It takes all 2 to get through Apple's notarization flow._

**Second, why is it necessary to notarize your packages?**

Notarization gives users more confidence that the Developer ID-signed software you distribute has been checked by Apple for malicious components. Notarization is not App Review. The Apple notary service is an automated system that scans your software for malicious content, checks for code-signing issues, and returns the results to you quickly.

Beginning in macOS 10.14.5, software signed with a new Developer ID certificate and all new or updated kernel extensions **must be notarized to run**. Beginning in macOS 10.15, all software built after June 1, 2019, and distributed with Developer ID must be notarized. 

_Note that you aren’t required to notarize software that you distribute through the Mac App Store because the App Store submission process already includes equivalent security checks._

## Useful references

- [Notarizing macOS Software Before Distribution](https://developer.apple.com/documentation/security/notarizing_macos_software_before_distribution)
- [Step by step to install an Apple certificate on a macOs runner (Github Actions)](https://docs.github.com/en/actions/deployment/deploying-xcode-applications/installing-an-apple-certificate-on-macos-runners-for-xcode-development)
- [Script to sign each file inside a folder](https://gist.github.com/GuillaumeFalourd/4efc73f1a6014b791c0ef223a023520a)
- [Script to generate a macos package](https://github.com/KosalaHerath/macos-installer-builder/tree/master/macOS-x64)
## Workflow example

This workflow shows how to perform the whole signature and notarization process of a `.pkg` package, when you don't directly save the `.p12` certificates as base64 secrets on your github repository. If you only have the `.cer` and `.key` certificate files, you can follow generate the `.p12` certificate file with `openssl`.

```yaml
jobs:
  [...] # Previous job generating the artifact with the application/software files.

  pkg:
    needs: [...]
    runs-on: macos-latest
    steps:
      - name: "Checkout"
        uses: actions/checkout@v4

      - name: "Download macos bin file"
        uses: actions/download-artifact@v4
        with:
          name: macos-bin-files
          path: dist/
      
      - name: "Generate signed pkg"
        run: |     
          ----- Create certificate files from secrets base64 -----
          echo ${{ secrets.DEVELOPER_ID_INSTALLER_CER }} | base64 --decode > certificate_installer.cer
          echo ${{ secrets.DEVELOPER_ID_INSTALLER_KEY }} | base64 --decode > certificate_installer.key
          echo ${{ secrets.DEVELOPER_ID_APPLICATION_CER }} | base64 --decode > certificate_application.cer
          echo ${{ secrets.DEVELOPER_ID_APPLICATION_KEY }} | base64 --decode > certificate_application.key
          
          ----- Create p12 file -----
          openssl pkcs12 -export -name zup -in certificate_installer.cer -inkey certificate_installer.key -passin pass:${{ secrets.KEY_PASSWORD }} -out certificate_installer.p12 -passout pass:${{ secrets.P12_PASSWORD }}
          openssl pkcs12 -export -name zup -in certificate_application.cer -inkey certificate_application.key -passin pass:${{ secrets.KEY_PASSWORD }} -out certificate_application.p12 -passout pass:${{ secrets.P12_PASSWORD }}
          
          ----- Configure Keychain -----
          KEYCHAIN_PATH=$RUNNER_TEMP/app-signing.keychain-db
          security create-keychain -p "${{ secrets.KEYCHAIN_PASSWORD }}" $KEYCHAIN_PATH
          security set-keychain-settings -lut 21600 $KEYCHAIN_PATH
          security unlock-keychain -p "${{ secrets.KEYCHAIN_PASSWORD }}" $KEYCHAIN_PATH
          
          ----- Import certificates on Keychain -----
          security import certificate_installer.p12 -P "${{ secrets.P12_PASSWORD }}" -A -t cert -f pkcs12 -k $KEYCHAIN_PATH
          security import certificate_application.p12 -P "${{ secrets.P12_PASSWORD }}" -A -t cert -f pkcs12 -k $KEYCHAIN_PATH
          security list-keychain -d user -s $KEYCHAIN_PATH
          
          ----- Codesign files with script -----
          use a script to sign each file from the artifact (ref: https://gist.github.com/GuillaumeFalourd/4efc73f1a6014b791c0ef223a023520a)
          
          ----- Generate PKG from codesigned files -----
          use a macos installer script (ref: https://github.com/KosalaHerath/macos-installer-builder/tree/master/macOS-x64)
          
          ----- Sign PKG file -----
          productsign --sign "${{ secrets.DEVELOPER_ID_INSTALLER_NAME }}" $INPUT_FILE_PATH $OUTPUT_FILE_PATH
          
          - name: "Notarize Release Build PKG"
            uses: devbotsxyz/xcode-notarize@v1 
            with:
              product-path: $PATH_TO_PKG
              appstore-connect-username: ${{ secrets.APPLE_ACCOUNT_USERNAME }}
              appstore-connect-password: ${{ secrets.APPLE_ACCOUNT_PASSWORD }}
              primary-bundle-id: 'BUNDLE_ID'
          
          - name: "Staple Release Build"
            uses: devbotsxyz/xcode-staple@v1
            with:
              product-path: $PATH_TO_PKG
  
  [...] # Next job to distribute the package
```

**Note that in this workflow example:**

- the `secrets.P12_PASSWORD` and the `secrets.KEYCHAIN_PASSWORD` can use any value (it would be different for the `P12_PASSWORD` value if you added directly the `.p12` file content as a base64 secret).
- the `secrets.KEY_PASSWORD` isn't mandatory (it may depends on the `.key` file).
- the `Developer Id certificates name` looks like this: `Org or User name (xxxxxxxxxx)`.
- the `BUNDLE_ID` can be any value.

**Note that the Notary tools have been updated on Apple.** 

Previously the action below was used in the process but has been removed from the marketplace:

```
- name: "Notarize Release Build PKG"
  uses: devbotsxyz/xcode-notarize@v1 
  with:
   product-path: $PATH_TO_PKG
   appstore-connect-username: ${{ secrets.APPLE_ACCOUNT_USERNAME }}
   appstore-connect-password: ${{ secrets.APPLE_ACCOUNT_PASSWORD }}
   primary-bundle-id: 'BUNDLE_ID'
```

I created [this one](https://github.com/GuillaumeFalourd/notary-tools) to substitute the notarize and staple actions:
```
- name: "Notarize and Staple Release Build"
  uses: GuillaumeFalourd/notary-tools@v1
  with:
    product_path: "$PATH_TO_PKG_OR_APP"
    apple_id: ${{ secrets.NOTARIZATION_USERNAME }}
    password: ${{ secrets.NOTARIZATION_PASSWORD }}
    team_id: ${{ secrets.NOTARIZATION_TEAMID }}
    staple: true
    xcode_path: '/Applications/Xcode.app'
```

## 🤝 Contribution

Would like to contribute to the repository? Here are the [guidelines](CONTRIBUTING.md) 🚀

<!-- 
<a href="https://github.com/GuillaumeFalourd/sign-and-notarize-gha/graphs/contributors">
  <img src="https://contrib.rocks/image?repo=GuillaumeFalourd/sign-and-notarize-gha" />
</a>

(Made with [contributors-img](https://contrib.rocks)) -->

#### If you have any question, feel free to open an issue, I'll try to help 🙂
