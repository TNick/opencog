
# We need to put this in a library, so that MOSESUTest can link to it.
ADD_LIBRARY (moses_exec
    moses_exec
    moses_exec_def
)

ADD_EXECUTABLE (moses-exec moses-exec)
TARGET_LINK_LIBRARIES (moses-exec
	moses_exec
	moses
)
SET_TARGET_PROPERTIES(moses-exec PROPERTIES OUTPUT_NAME "moses")

ADD_EXECUTABLE (eval-diversity eval-diversity)
TARGET_LINK_LIBRARIES (eval-diversity
    moses
    cogutil
    ${Boost_PROGRAM_OPTIONS_LIBRARY} 
)

ADD_EXECUTABLE (eval-candidate eval-candidate)
TARGET_LINK_LIBRARIES (eval-candidate
    moses
    cogutil
    ${Boost_PROGRAM_OPTIONS_LIBRARY}
)

# install moses-exec
INSTALL(TARGETS moses-exec eval-diversity eval-candidate RUNTIME DESTINATION bin)

#install header files
INSTALL(FILES
	moses_exec.h
	moses_exec_def.h

	DESTINATION

	"include/${PROJECT_NAME}/learning/moses/main" 
)
