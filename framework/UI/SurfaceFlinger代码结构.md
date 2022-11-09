# Binder

  **接口 ** frameworks/native/libs/gui/include/gui/ISurfaceComposer.h:
  
  **客户端：** frameworks/native/libs/gui/SurfaceComposerClient.cpp 
  
  **服务端：** SurfaceFlinger.cpp
  
# 接口定义
  
 frameworks/native/libs/gui/include/gui/ISurfaceComposer.h
 
 ```
 class ISurfaceComposer: public IInterface {
 public:
    DECLARE_META_INTERFACE(SurfaceComposer)
    
 };
 
 class BnSurfaceComposer: public BnInterface<ISurfaceComposer> {
 public:
   virtual status_t onTransact(uint32_t code, const Parcel& data,
            Parcel* reply, uint32_t flags = 0);
 };

 ```
 
 frameworks/native/libs/gui/ISurfaceComposer.cpp
 
 ```
 class BpSurfaceComposer : public BpInterface<ISurfaceComposer>
{
}
 ```
 
 # 服务端
 
 frameworks/native/services/surfaceflinger/SurfaceFlinger.h 
 
 ```
 class SurfaceFlinger : public BnSurfaceComposer,
                       public PriorityDumper,
                       public ClientCache::ErasedRecipient,
                       private IBinder::DeathRecipient,
                       private HWC2::ComposerCallback {
 }
 ```

frameworks/native/services/surfaceflinger/main_surfaceflinger.cpp

```
int main(int, char**) {
  sp<SurfaceFlinger> flinger = surfaceflinger::createSurfaceFlinger();
  sm->addService(String16(SurfaceFlinger::getServiceName()), flinger, false,
                   IServiceManager::DUMP_FLAG_PRIORITY_CRITICAL | IServiceManager::DUMP_FLAG_PROTO);
}
```
 # 客户端

frameworks/native/libs/gui/SurfaceComposerClient.cpp

```
void SurfaceComposerClient::onFirstRef() {
    sp<ISurfaceComposer> sf(ComposerService::getComposerService());
    if (sf != nullptr && mStatus == NO_INIT) {
        sp<ISurfaceComposerClient> conn;
        conn = sf->createConnection();
        if (conn != nullptr) {
            mClient = conn;
            mStatus = NO_ERROR;
        }
    }
}
```
