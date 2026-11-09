# Unified Tinode Application

এই ফোল্ডারে ত্রietro প্রজেক্টের সোর্স কোড যৌক্তিকভাবে সাজানো আছে:

- `backend/` — Go ব্যাকএন্ড সার্ভার (`chat-master` থেকে)
- `android/` — অ্যান্ড্রয়েড নেটিভ 앱 (`tindroid-master` থেকে)
- `web/` — রিয়াক্ট ওয়েব ক্লায়েন্ট (`webapp-master` থেকে)
- `shared/` — যৌথ ডকুমেন্টেশন ও অ্যাসেটস

## ফোল্ডার কাঠামো

```
unified-project/
├── backend/          # Tinode Go Server (0.25)
│   ├── server/       # Core server code
│   ├── docker/       # Docker deployment
│   └── ...
├── android/          # Android Native Client
│   ├── app/          # Android application
│   ├── tinodesdk/    # Tinode Android SDK
│   └── ...
├── web/              # React Web Client
│   ├── src/          # React source code
│   ├── umd/          # Built UMD bundles
│   └── ...
└── shared/           # Shared docs & assets
    └── docs/
```

## মূল ফাইলগুলো preserved

মূল এক্সট্রাক্ট করা ফোল্ডারগুলো (`chat-master/`, `tindroid-master/`, `webapp-master/`) এবং জিপ ফাইলগুলো (`chat-master.zip`, `tindroid-master.zip`, `webapp-master.zip`) রেপোর রুটে অখণ্ড অবস্থানে রয়েছে। আপনি নিশ্চিত হলে সেগুলো ডিলিট করতে পারেন।

## বিল্ড ইন্সট্রাকশন

বिस्तারিত বিল্ড ও ডেপ্লয়মেন্ট গাইডের জন্য দেখুন:
- `GOOGLE_AI_STUDIO_HANDOFF.md` (গুগল এআই স্টুডিও হ্যান্ডওফ প্যাকেজ)
- `UNIFIED_ARCHITECTURE.md` (ইউনিফাইড আর্কিটেকচার ও রোডম্যাপ)

## লাইসেন্স

মূল প্রজেক্ট লাইসেন্স প্রযুয়্য (GPL 3.0 সার্ভারের জন্য, Apache 2.0 ক্লায়েন্টদের জন্য).