# Don't Forget

Critical items to remember for next versions and future development.

## YoloWrap - v2.0 System-Level Service Integration

**Current State (v1.0):**
- ✅ API runs as Windows Service via NSSM
- ✅ Uses user account (jmwan) to access conda environment
- ✅ Fully functional and production-ready
- ✅ Auto-starts on reboot
- ✅ Auto-restarts on failure

**Next Version TODO:**
- [ ] **Move to System-Level Service** - Don't require user account credentials
  - Create standalone Python executable bundle (PyInstaller)
  - Build proper Windows Service wrapper (not NSSM-dependent)
  - Use environment variables instead of user-specific paths
  - Enable system-wide installation without user context

**Why This Matters:**
- Current setup requires user credentials in NSSM configuration
- **CRITICAL**: Service will BREAK if user password changes
  - To fix: Must reconfigure NSSM with new password:
    - `nssm.exe set YoloWrapAPI ObjectName ".\jmwan" "NewPassword"`
    - Then restart service
- Can't scale to multi-user or server deployments
- NSSM is third-party dependency

**Implementation Notes:**
- See: `c:\Users\jmwan\MarengoCam\YoloWrap\README.md` - "Future Improvements (v2.0)"
- See: `c:\Users\jmwan\MarengoCam\YoloWrap\WINDOWS_SERVICE_SETUP.md` - Full service setup docs
- See: `c:\Users\jmwan\MarengoCam\YoloWrap\API_STARTUP_GUIDE.md` - Startup documentation
