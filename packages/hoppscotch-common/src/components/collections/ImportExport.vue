<template>
  <ImportExportBase
    ref="collections-import-export"
    modal-title="modal.collections"
    :importer-modules="importerModules"
    :exporter-modules="exporterModules"
    @hide-modal="emit('hide-modal')"
  />
</template>

<script setup lang="ts">
import { HoppCollection } from "@hoppscotch/data"
import * as E from "fp-ts/Either"
import { PropType, computed, ref } from "vue"

import { FileSource } from "~/helpers/import-export/import/import-sources/FileSource"
import { UrlSource } from "~/helpers/import-export/import/import-sources/UrlSource"

import IconFile from "~icons/lucide/file"

import {
  hoppRESTImporter,
  hoppInsomniaImporter,
  hoppPostmanImporter,
  toTeamsImporter,
  hoppOpenAPIImporter,
  harImporter,
} from "~/helpers/import-export/import/importers"

import { defineStep } from "~/composables/step-components"

import { useI18n } from "~/composables/i18n"
import { useToast } from "~/composables/toast"
import { appendRESTCollections, restCollections$ } from "~/newstore/collections"
import MyCollectionImport from "~/components/importExport/ImportExportSteps/MyCollectionImport.vue"

import IconFolderPlus from "~icons/lucide/folder-plus"
import IconOpenAPI from "~icons/lucide/file"
import IconPostman from "~icons/hopp/postman"
import IconInsomnia from "~icons/hopp/insomnia"
import IconGithub from "~icons/lucide/github"
import IconLink from "~icons/lucide/link"

import IconUser from "~icons/lucide/user"
import { useReadonlyStream } from "~/composables/stream"

import { getTeamCollectionJSON } from "~/helpers/backend/helpers"

import { platform } from "~/platform"

import { initializeDownloadFile } from "~/helpers/import-export/export"
import { gistExporter } from "~/helpers/import-export/export/gist"
import { myCollectionsExporter } from "~/helpers/import-export/export/myCollections"
import { teamCollectionsExporter } from "~/helpers/import-export/export/teamCollections"

import { GistSource } from "~/helpers/import-export/import/import-sources/GistSource"
import { ImporterOrExporter } from "~/components/importExport/types"
import { TeamWorkspace } from "~/services/workspace.service"

const t = useI18n()
const toast = useToast()

type CollectionType =
  | {
      type: "team-collections"
      selectedTeam: TeamWorkspace
    }
  | { type: "my-collections" }

const props = defineProps({
  collectionsType: {
    type: Object as PropType<CollectionType>,
    default: () => ({
      type: "my-collections",
      selectedTeam: undefined,
    }),
    required: true,
  },
})

const currentUser = useReadonlyStream(
  platform.auth.getCurrentUserStream(),
  platform.auth.getCurrentUser()
)

const myCollections = useReadonlyStream(restCollections$, [])

const showImportFailedError = () => {
  toast.error(t("import.failed"))
}

const handleImportToStore = async (collections: HoppCollection[]) => {
  const importResult =
    props.collectionsType.type === "my-collections"
      ? await importToPersonalWorkspace(collections)
      : await importToTeamsWorkspace(collections)

  if (E.isRight(importResult)) {
    toast.success(t("state.file_imported"))
    emit("hide-modal")
  } else {
    toast.error(t("import.failed"))
  }
}

const importToPersonalWorkspace = (collections: HoppCollection[]) => {
  appendRESTCollections(collections)
  return E.right({
    success: true,
  })
}

function translateToTeamCollectionFormat(x: HoppCollection) {
  const folders: HoppCollection[] = (x.folders ?? []).map(
    translateToTeamCollectionFormat
  )

  const data = {
    auth: x.auth,
    headers: x.headers,
  }

  const obj = {
    ...x,
    folders,
    data,
  }

  if (x.id) obj.id = x.id

  return obj
}

const importToTeamsWorkspace = async (collections: HoppCollection[]) => {
  if (!hasTeamWriteAccess.value || !selectedTeamID.value) {
    return E.left({
      success: false,
    })
  }

  const transformedCollection = collections.map((collection) =>
    translateToTeamCollectionFormat(collection)
  )

  const res = await toTeamsImporter(
    JSON.stringify(transformedCollection),
    selectedTeamID.value
  )()

  return E.isRight(res)
    ? E.right({ success: true })
    : E.left({
        success: false,
      })
}

const emit = defineEmits<{
  (e: "hide-modal"): () => void
}>()

const isHoppMyCollectionExporterInProgress = ref(false)
const isHoppTeamCollectionExporterInProgress = ref(false)
const isHoppGistCollectionExporterInProgress = ref(false)

const isTeamWorkspace = computed(() => {
  return props.collectionsType.type === "team-collections"
})

const HoppRESTImporter: ImporterOrExporter = {
  metadata: {
    id: "hopp_rest",
    name: "import.from_json",
    title: "import.from_json_description",
    icon: IconFolderPlus,
    disabled: false,
    applicableTo: ["personal-workspace", "team-workspace", "url-import"],
  },
  component: FileSource({
    caption: "import.from_file",
    acceptedFileTypes: ".json",
    onImportFromFile: async (content) => {
      const res = await hoppRESTImporter(content)()

      if (E.isRight(res)) {
        handleImportToStore(res.right)

        platform.analytics?.logEvent({
          type: "HOPP_IMPORT_COLLECTION",
          importer: "import.from_json",
          platform: "rest",
          workspaceType: isTeamWorkspace.value ? "team" : "personal",
        })
      } else {
        showImportFailedError()
      }
    },
  }),
}

const HoppMyCollectionImporter: ImporterOrExporter = {
  metadata: {
    id: "hopp_my_collection",
    name: "import.from_my_collections",
    title: "import.from_my_collections_description",
    icon: IconUser,
    disabled: false,
    applicableTo: ["team-workspace"],
  },
  component: defineStep("my_collection_import", MyCollectionImport, () => ({
    async onImportFromMyCollection(content) {
      handleImportToStore([content])

      // our analytics consider this as an export event, so keeping compatibility with that
      platform.analytics?.logEvent({
        type: "HOPP_EXPORT_COLLECTION",
        exporter: "import_to_teams",
        platform: "rest",
      })
    },
  })),
}

const HoppOpenAPIImporter: ImporterOrExporter = {
  metadata: {
    id: "hopp_openapi",
    name: "import.from_openapi",
    title: "import.from_openapi_description",
    icon: IconOpenAPI,
    disabled: false,
    applicableTo: ["personal-workspace", "team-workspace", "url-import"],
  },
  supported_sources: [
    {
      id: "file_import",
      name: "import.from_file",
      icon: IconFile,
      step: FileSource({
        caption: "import.from_file",
        acceptedFileTypes: ".json, .yaml, .yml",
        onImportFromFile: async (content) => {
          const res = await hoppOpenAPIImporter(content)()

          if (E.isRight(res)) {
            handleImportToStore(res.right)

            platform.analytics?.logEvent({
              platform: "rest",
              type: "HOPP_IMPORT_COLLECTION",
              importer: "import.from_openapi",
              workspaceType: isTeamWorkspace.value ? "team" : "personal",
            })
          } else {
            showImportFailedError()
          }
        },
      }),
    },
    {
      id: "url_import",
      name: "import.from_url",
      icon: IconLink,
      step: UrlSource({
        caption: "import.from_url",
        onImportFromURL: async (content) => {
          const res = await hoppOpenAPIImporter([content])()

          if (E.isRight(res)) {
            handleImportToStore(res.right)

            platform.analytics?.logEvent({
              platform: "rest",
              type: "HOPP_IMPORT_COLLECTION",
              importer: "import.from_openapi",
              workspaceType: isTeamWorkspace.value ? "team" : "personal",
            })
          } else {
            showImportFailedError()
          }
        },
      }),
    },
  ],
}

const HoppPostmanImporter: ImporterOrExporter = {
  metadata: {
    id: "hopp_postman",
    name: "import.from_postman",
    title: "import.from_postman_description",
    icon: IconPostman,
    disabled: false,
    applicableTo: ["personal-workspace", "team-workspace", "url-import"],
  },
  component: FileSource({
    caption: "import.from_file",
    acceptedFileTypes: ".json",
    onImportFromFile: async (content) => {
      const res = await hoppPostmanImporter(content)()

      if (E.isRight(res)) {
        handleImportToStore(res.right)

        platform.analytics?.logEvent({
          platform: "rest",
          type: "HOPP_IMPORT_COLLECTION",
          importer: "import.from_postman",
          workspaceType: isTeamWorkspace.value ? "team" : "personal",
        })
      } else {
        showImportFailedError()
      }
    },
  }),
}

const HoppInsomniaImporter: ImporterOrExporter = {
  metadata: {
    id: "hopp_insomnia",
    name: "import.from_insomnia",
    title: "import.from_insomnia_description",
    icon: IconInsomnia,
    disabled: true,
    applicableTo: ["personal-workspace", "team-workspace", "url-import"],
  },
  component: FileSource({
    caption: "import.from_file",
    acceptedFileTypes: ".json",
    onImportFromFile: async (content) => {
      const res = await hoppInsomniaImporter(content)()

      if (E.isRight(res)) {
        handleImportToStore(res.right)

        platform.analytics?.logEvent({
          platform: "rest",
          type: "HOPP_IMPORT_COLLECTION",
          importer: "import.from_insomnia",
          workspaceType: isTeamWorkspace.value ? "team" : "personal",
        })
      } else {
        showImportFailedError()
      }
    },
  }),
}

const HoppGistImporter: ImporterOrExporter = {
  metadata: {
    id: "hopp_gist",
    name: "import.from_gist",
    title: "import.from_gist_description",
    icon: IconGithub,
    disabled: true,
    applicableTo: ["personal-workspace", "team-workspace", "url-import"],
  },
  component: GistSource({
    caption: "import.from_url",
    onImportFromGist: async (content) => {
      if (E.isLeft(content)) {
        showImportFailedError()
        return
      }

      const res = await hoppRESTImporter(content.right)()

      if (E.isRight(res)) {
        handleImportToStore(res.right)

        platform.analytics?.logEvent({
          platform: "rest",
          type: "HOPP_IMPORT_COLLECTION",
          importer: "import.from_gist",
          workspaceType: isTeamWorkspace.value ? "team" : "personal",
        })
      } else {
        showImportFailedError()
      }
    },
  }),
}

const HoppMyCollectionsExporter: ImporterOrExporter = {
  metadata: {
    id: "hopp_my_collections",
    name: "export.as_json",
    title: "action.download_file",
    icon: IconUser,
    disabled: false,
    applicableTo: ["personal-workspace"],
    isLoading: isHoppMyCollectionExporterInProgress,
  },
  action: async () => {
    if (!myCollections.value.length) {
      return toast.error(t("error.no_collections_to_export"))
    }

    isHoppMyCollectionExporterInProgress.value = true

    const message = await initializeDownloadFile(
      myCollectionsExporter(myCollections.value),
      "hoppscotch-personal-collections"
    )

    if (E.isRight(message)) {
      toast.success(t("state.download_started"))

      platform.analytics?.logEvent({
        type: "HOPP_EXPORT_COLLECTION",
        exporter: "json",
        platform: "rest",
      })
    } else {
      toast.error(t(message.left))
    }

    isHoppMyCollectionExporterInProgress.value = false
  },
}

const HoppTeamCollectionsExporter: ImporterOrExporter = {
  metadata: {
    id: "hopp_team_collections",
    name: "export.as_json",
    title: "export.as_json",
    icon: IconUser,
    disabled: false,
    applicableTo: ["team-workspace"],
    isLoading: isHoppTeamCollectionExporterInProgress,
  },
  action: async () => {
    isHoppTeamCollectionExporterInProgress.value = true
    if (
      props.collectionsType.type === "team-collections" &&
      props.collectionsType.selectedTeam
    ) {
      const res = await teamCollectionsExporter(
        props.collectionsType.selectedTeam.teamID
      )

      if (E.isRight(res)) {
        const message = await initializeDownloadFile(
          res.right,
          "hoppscotch-team-collections"
        )

        E.isRight(message)
          ? toast.success(t(message.right))
          : toast.error(t(message.left))

        platform.analytics?.logEvent({
          type: "HOPP_EXPORT_COLLECTION",
          exporter: "json",
          platform: "rest",
        })
      } else {
        toast.error(res.left)
      }
    }

    isHoppTeamCollectionExporterInProgress.value = false
  },
}

const HoppGistCollectionsExporter: ImporterOrExporter = {
  metadata: {
    id: "create_secret_gist",
    name: "export.create_secret_gist",
    icon: IconGithub,
    disabled: !currentUser.value
      ? true
      : currentUser.value?.provider !== "github.com",
    title:
      // eslint-disable-next-line @typescript-eslint/ban-ts-comment
      // @ts-ignore
      currentUser?.value?.provider === "github.com"
        ? "export.create_secret_gist_tooltip_text"
        : "export.require_github",
    applicableTo: ["personal-workspace", "team-workspace"],
    isLoading: isHoppGistCollectionExporterInProgress,
  },
  action: async () => {
    isHoppGistCollectionExporterInProgress.value = true

    const collectionJSON = await getCollectionJSON()
    const accessToken = currentUser.value?.accessToken

    if (!accessToken) {
      toast.error(t("error.something_went_wrong"))
      isHoppGistCollectionExporterInProgress.value = false
      return
    }

    if (E.isRight(collectionJSON)) {
      const res = await gistExporter(collectionJSON.right, accessToken)

      if (E.isLeft(res)) {
        toast.error(t("export.failed"))
        return
      }

      toast.success(t("export.secret_gist_success"))

      platform.analytics?.logEvent({
        type: "HOPP_EXPORT_COLLECTION",
        exporter: "gist",
        platform: "rest",
      })

      platform.io.openExternalLink(res.right)
    } else {
      toast.error(collectionJSON.left)
    }

    isHoppGistCollectionExporterInProgress.value = false
  },
}

const HARImporter: ImporterOrExporter = {
  metadata: {
    id: "har",
    name: "import.from_har",
    title: "import.from_har_description",
    icon: IconFile,
    disabled: false,
    applicableTo: ["personal-workspace", "team-workspace"],
  },
  component: FileSource({
    caption: "import.from_file",
    acceptedFileTypes: ".har",
    onImportFromFile: async (content) => {
      const res = await harImporter(content)

      if (E.isLeft(res)) {
        showImportFailedError()
        return
      }

      handleImportToStore(res.right)

      platform.analytics?.logEvent({
        type: "HOPP_IMPORT_COLLECTION",
        importer: "import.from_har",
        platform: "rest",
        workspaceType: isTeamWorkspace.value ? "team" : "personal",
      })
    },
  }),
}

const importerModules = computed(() => {
  const enabledImporters = [
    HoppRESTImporter,
    HoppMyCollectionImporter,
    HoppOpenAPIImporter,
    HoppPostmanImporter,
    HoppInsomniaImporter,
    HoppGistImporter,
    HARImporter,
  ]

  const isTeams = props.collectionsType.type === "team-collections"

  return enabledImporters.filter((importer) => {
    return isTeams
      ? importer.metadata.applicableTo.includes("team-workspace")
      : importer.metadata.applicableTo.includes("personal-workspace")
  })
})

const exporterModules = computed(() => {
  const enabledExporters = [
    HoppMyCollectionsExporter,
    HoppTeamCollectionsExporter,
  ]

  if (platform.platformFeatureFlags.exportAsGIST) {
    enabledExporters.push(HoppGistCollectionsExporter)
  }

  return enabledExporters.filter((exporter) => {
    return exporter.metadata.applicableTo.includes(
      props.collectionsType.type === "my-collections"
        ? "personal-workspace"
        : "team-workspace"
    )
  })
})

const hasTeamWriteAccess = computed(() => {
  const { collectionsType } = props

  const isTeamCollection = collectionsType.type === "team-collections"

  if (!isTeamCollection || !collectionsType.selectedTeam) {
    return false
  }

  return (
    collectionsType.selectedTeam.role === "EDITOR" ||
    collectionsType.selectedTeam.role === "OWNER"
  )
})

const selectedTeamID = computed(() => {
  const { collectionsType } = props

  return collectionsType.type === "team-collections"
    ? collectionsType.selectedTeam?.teamID
    : undefined
})

const getCollectionJSON = async () => {
  if (
    props.collectionsType.type === "team-collections" &&
    props.collectionsType.selectedTeam?.teamID
  ) {
    const res = await getTeamCollectionJSON(
      props.collectionsType.selectedTeam?.teamID
    )

    return E.isRight(res) ? E.right(res.right) : E.left(res.left)
  }

  if (props.collectionsType.type === "my-collections") {
    return E.right(JSON.stringify(myCollections.value, null, 2))
  }

  return E.left("INVALID_SELECTED_TEAM_OR_INVALID_COLLECTION_TYPE")
}
</script>
