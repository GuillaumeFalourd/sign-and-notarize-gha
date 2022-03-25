# sign-and-notarize-gha

Repository explaining step by step how to sign and notarize packages with Apple using Github Actions

## Premisses

- [ ] ON GOING

## Explanaition

- [ ] ON GOING

## Workflow example

```yaml
jobs:
  [...]

  pkg:
    needs: [...]
    runs-on: macos-latest
    steps:
      - name: "Checkout"
        uses: actions/checkout@v2

      - name: "Download macos bin file"
        uses: actions/download-artifact@v2
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
  [...]
```

## 🤝 Contribution

Would like to contribute to the repository? Here are the [guidelines](CONTRIBUTING.md) 🚀

<!-- 
<a href="https://github.com/GuillaumeFalourd/sign-and-notarize-gha/graphs/contributors">
  <img src="https://contrib.rocks/image?repo=GuillaumeFalourd/sign-and-notarize-gha" />
</a>

(Made with [contributors-img](https://contrib.rocks)) -->