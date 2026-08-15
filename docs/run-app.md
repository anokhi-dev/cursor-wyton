cd /Users/admin/Documents/techfidants/wyton-org/wyton-app/ios
rm -rf Pods Podfile.lock build
pod install --repo-update
cd ..
rm -rf ~/Library/Developer/Xcode/DerivedData/wyton-*
npm run ios -- --simulator="iPhone 17 Pro"