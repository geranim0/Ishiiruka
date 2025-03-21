set(GUI_SRCS
	AboutDolphin.cpp
	ControllerConfigDiag.cpp
	Cheats/ActionReplayCodesPanel.cpp
	Cheats/ARCodeAddEdit.cpp
	Cheats/CheatSearchTab.cpp
	Cheats/CheatsWindow.cpp
	Cheats/CreateCodeDialog.cpp
	Cheats/GeckoCodeDiag.cpp
	Config/AdvancedConfigPane.cpp
	Config/AudioConfigPane.cpp
	Config/ConfigMain.cpp
	Config/SlippiConfigPane.cpp
	Config/GameCubeConfigPane.cpp
	Config/GCAdapterConfigDiag.cpp
	Config/GeneralConfigPane.cpp
	Config/InterfaceConfigPane.cpp
	Config/PathConfigPane.cpp
	Config/WiiConfigPane.cpp
	Debugger/BreakpointDlg.cpp
	Debugger/BreakpointView.cpp
	Debugger/BreakpointWindow.cpp
	Debugger/CodeView.cpp
	Debugger/CodeWindow.cpp
	Debugger/CodeWindowFunctions.cpp
	Debugger/DSPDebugWindow.cpp
	Debugger/DSPRegisterView.cpp
	Debugger/DebuggerPanel.cpp
	Debugger/DebuggerUIUtil.cpp
	Debugger/JitWindow.cpp
	Debugger/MemoryCheckDlg.cpp
	Debugger/MemoryView.cpp
	Debugger/MemoryWindow.cpp
	Debugger/RegisterView.cpp
	Debugger/RegisterWindow.cpp
	Debugger/WatchView.cpp
	Debugger/WatchWindow.cpp
	ISOProperties/FileSystemPanel.cpp
	ISOProperties/InfoPanel.cpp
	ISOProperties/ISOProperties.cpp
	NetPlay/ChangeGameDialog.cpp
	NetPlay/MD5Dialog.cpp
	NetPlay/NetPlayLauncher.cpp
	NetPlay/NetPlaySetupFrame.cpp
	NetPlay/NetWindow.cpp
	NetPlay/PadMapDialog.cpp
	Input/InputConfigDiag.cpp
	Input/InputConfigDiagBitmaps.cpp
	Input/HotkeyInputConfigDiag.cpp
	Input/GCPadInputConfigDiag.cpp
	Input/MicButtonConfigDiag.cpp
	Input/GCKeyboardInputConfigDiag.cpp
	Input/WiimoteInputConfigDiag.cpp
	Input/NunchukInputConfigDiag.cpp
	Input/ClassicInputConfigDiag.cpp
	Input/GuitarInputConfigDiag.cpp
	Input/DrumsInputConfigDiag.cpp
	Input/TurntableInputConfigDiag.cpp
	DolphinSlider.cpp
	FifoPlayerDlg.cpp
	Frame.cpp
	FrameAui.cpp
	FrameTools.cpp
	GameListCtrl.cpp
	ISOFile.cpp
	LogConfigWindow.cpp
	LogWindow.cpp
	Main.cpp
	MainMenuBar.cpp
	PlaybackSlider.cpp
	MainToolBar.cpp
	MemcardManager.cpp
	PatchAddEdit.cpp
	PostProcessingConfigDiag.cpp
	SoftwareVideoConfigDialog.cpp
	TASInputDlg.cpp
	VideoConfigDiag.cpp
	WxEventUtils.cpp
	WXInputBase.cpp
	WxUtils.cpp)

set(NOGUI_SRCS Main.cpp MainNoGUI.cpp)

if(USE_X11)
	set(GUI_SRCS ${GUI_SRCS} X11Utils.cpp)
	set(NOGUI_SRCS ${NOGUI_SRCS} X11Utils.cpp)
endif()

set(WXLIBS ${wxWidgets_LIBRARIES})

if(${CMAKE_SYSTEM_NAME} STREQUAL "Linux")
	set(WXLIBS ${WXLIBS} dl)
endif()

list(APPEND LIBS core uicommon)

if(APPLE)
	if(wxWidgets_FOUND)
		list(APPEND WXLIBS
			${APPKIT_LIBRARY}
			${APPSERV_LIBRARY}
			${ATB_LIBRARY}
			${CARBON_LIBRARY}
			${COCOA_LIBRARY}
			${COREFUND_LIBRARY}
			${CORESERV_LIBRARY}
			${IOK_LIBRARY}
			${OPENGL_LIBRARY}
			)
	endif()

	# Add resource files to application bundle.
	set(RESOURCES resources/Dolphin.icns)
	list(APPEND SRCS ${RESOURCES})
	set_source_files_properties(${RESOURCES} PROPERTIES
		MACOSX_PACKAGE_LOCATION Resources)
endif()

if(APPLE)
	set(DOLPHIN_EXE_BASE "Slippi Dolphin")
else()
	set(DOLPHIN_EXE_BASE dolphin-emu)
endif()

set(DOLPHIN_EXE ${DOLPHIN_EXE_BASE})

include(FindGettext)
if(GETTEXT_MSGMERGE_EXECUTABLE AND GETTEXT_MSGFMT_EXECUTABLE AND wxWidgets_FOUND)
	file(GLOB LINGUAS ${CMAKE_SOURCE_DIR}/Languages/po/*.po)
	add_custom_target(translations ALL)
	GETTEXT_CREATE_TRANSLATIONS(${CMAKE_SOURCE_DIR}/Languages/po/dolphin-emu.pot ${LINGUAS})
endif()

if(wxWidgets_FOUND)
	add_executable("${DOLPHIN_EXE}" ${SRCS} ${GUI_SRCS})
	target_link_libraries("${DOLPHIN_EXE}" ${LIBS} ${WXLIBS})

	# Link SlippiRustExtensions in.
	#
	# (This doesn't feel perfect here, but truth be told I can't think offhand of
	# a better location for it at the moment.)
	set(RUST_FEATURES "")
	if(DIS_PLAYBACK)
		set(RUST_FEATURES "playback")
	endif()

	corrosion_import_crate(MANIFEST_PATH ${CMAKE_SOURCE_DIR}/Externals/SlippiRustExtensions/Cargo.toml ${RUST_FEATURES})
	target_link_libraries("${DOLPHIN_EXE}" PUBLIC slippi-rust-extensions)

	if(APPLE)
		include(BundleUtilities)
		set(BUNDLE_PATH ${CMAKE_RUNTIME_OUTPUT_DIRECTORY}/"${DOLPHIN_EXE}".app)
		string(REPLACE "\"" "" BUNDLE_PATH_NO_QUOTES ${BUNDLE_PATH})

		# Ask for an application bundle.
		set_target_properties(${DOLPHIN_EXE} PROPERTIES
			MACOSX_BUNDLE true
			MACOSX_BUNDLE_INFO_PLIST ${CMAKE_CURRENT_SOURCE_DIR}/Info.plist.in
		)
		
		# Link MoltenVK for the Vulkan (experimental) backend; the framework is
		# added into the bundle in Tools/deploy-mac.py due to legacy build reasons.
		find_library(MOLTENVK libvulkan.dylib PATHS ${CMAKE_SOURCE_DIR}/Externals/MoltenVK)
		message("MoltenVK Path: ${MOLTENVK}")
		target_link_libraries("${DOLPHIN_EXE}" ${MOLTENVK})

		# get rid of any old copies
		file (REMOVE_RECURSE ${BUNDLE_PATH}/Contents/Resources/Sys)
		if(NOT SKIP_POSTPROCESS_BUNDLE)
			# Fix up the bundle after it is finished.
			# There does not seem to be an easy way to run CMake commands post-build,
			# so we invoke CMake again on a generated script.
			file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/postprocess_bundle.cmake "
				include(BundleUtilities)
				message(\"Fixing up application bundle: ${BUNDLE_PATH_NO_QUOTES}\")
				message(\"(Note: This is only necessary to produce a redistributable binary.\")
				message(\"To skip, pass -DSKIP_POSTPROCESS_BUNDLE=1 to cmake.)\")
				set(BU_CHMOD_BUNDLE_ITEMS ON)
				execute_process(
					COMMAND ${CMAKE_SOURCE_DIR}/Tools/deploy-mac.py \"${BUNDLE_PATH_NO_QUOTES}\"
					RESULT_VARIABLE retcode
				)
				if(NOT \${retcode} EQUAL 0)
					message(FATAL_ERROR \"Error when postprocessing bundle (return code: \${retcode}).\")
				endif()
				file(INSTALL \"${CMAKE_SOURCE_DIR}/Data/Sys\"
					DESTINATION \"${BUNDLE_PATH_NO_QUOTES}/Contents/Resources\"
					)
				")
			add_custom_command(TARGET ${DOLPHIN_EXE} POST_BUILD
				COMMAND ${CMAKE_COMMAND} -P postprocess_bundle.cmake
				)
		else()
			add_custom_command(OUTPUT ${BUNDLE_PATH}/Contents/Resources/Sys
				COMMAND ln -nfs ${CMAKE_SOURCE_DIR}/Data/Sys ${BUNDLE_PATH}/Contents/Resources/Sys
				VERBATIM
				)
			add_custom_target(CopyDataIntoBundleWx ALL
				DEPENDS ${BUNDLE_PATH}/Contents/Resources/Sys
				)
		endif()

		if(GETTEXT_MSGMERGE_EXECUTABLE AND GETTEXT_MSGFMT_EXECUTABLE AND wxWidgets_FOUND)
			file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/copy_translations_into_bundle.cmake "
				file(GLOB TRANSLATION_FILES RELATIVE ${CMAKE_CURRENT_BINARY_DIR}
					${CMAKE_CURRENT_BINARY_DIR}/*.gmo
					)
				foreach(TRANSLATION_FILE \${TRANSLATION_FILES})
					string(REPLACE \".gmo\" \".lproj\" TRANSLATION_DIR
						\${TRANSLATION_FILE}
						)
					# It would be better to copy to the new name as a single action,
					# but I can't figure out a way to let CMake do that.
					file(COPY ${CMAKE_CURRENT_BINARY_DIR}/\${TRANSLATION_FILE}
						DESTINATION \"${BUNDLE_PATH_NO_QUOTES}/Contents/Resources/\${TRANSLATION_DIR}\"
						NO_SOURCE_PERMISSIONS
						)
					file(RENAME
						\"${BUNDLE_PATH_NO_QUOTES}/Contents/Resources/\${TRANSLATION_DIR}/\${TRANSLATION_FILE}\"
						\"${BUNDLE_PATH_NO_QUOTES}/Contents/Resources/\${TRANSLATION_DIR}/dolphin-emu.mo\"
						)
				endforeach(TRANSLATION_FILE)
				")

			file(GLOB PO_FILES RELATIVE ${CMAKE_SOURCE_DIR}/Languages/po
				${CMAKE_SOURCE_DIR}/Languages/po/*.po
				)
			string(REPLACE .po .gmo GMO_FILES "${PO_FILES}")

			add_custom_command(OUTPUT ${BUNDLE_PATH}/Contents/Resources/en.lproj
				COMMAND ${CMAKE_COMMAND} -P copy_translations_into_bundle.cmake
				DEPENDS ${GMO_FILES}
				${CMAKE_SOURCE_DIR}/Data/Sys
				VERBATIM
				)
			add_custom_target(CopyTranslationsIntoBundleWx ALL
				DEPENDS ${BUNDLE_PATH}/Contents/Resources/en.lproj
				)
		endif()

		if(MACOS_CODE_SIGNING)
			# Disable this as we need to codesign AFTER resources replacement.
			#add_custom_command(TARGET ${DOLPHIN_EXE}
			#	POST_BUILD COMMAND
			#	/usr/bin/codesign -f -s "${MACOS_CODE_SIGNING_IDENTITY}" --deep --options runtime --entitlements ${CMAKE_SOURCE_DIR}/Source/Core/DolphinWX/Entitlements.plist "${BUNDLE_PATH_NO_QUOTES}")
		endif()
	else()
		install(TARGETS ${DOLPHIN_EXE} RUNTIME DESTINATION ${bindir})
	endif()

	set(CPACK_PACKAGE_EXECUTABLES ${CPACK_PACKAGE_EXECUTABLES} ${DOLPHIN_EXE})
endif()

# Only build -nogui binaries when ENABLE_HEADLESS is on
if(USE_X11 AND ENABLE_HEADLESS)
	set(DOLPHIN_NOGUI_EXE ${DOLPHIN_EXE_BASE}-nogui)
	add_executable(${DOLPHIN_NOGUI_EXE} ${SRCS} ${NOGUI_SRCS})
	target_link_libraries(${DOLPHIN_NOGUI_EXE} ${LIBS} ${WXLIBS})
	set(CPACK_PACKAGE_EXECUTABLES ${CPACK_PACKAGE_EXECUTABLES} ${DOLPHIN_NOGUI_EXE})
	install(TARGETS ${DOLPHIN_NOGUI_EXE} RUNTIME DESTINATION ${bindir})
endif()

find_package(Boost)
if(Boost_FOUND)
  include_directories(${Boost_INCLUDE_DIRS})
  target_link_libraries("${DOLPHIN_EXE}" ${Boost_LIBRARIES} rt pthread)
endif()