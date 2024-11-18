FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443

FROM --platform=$BUILDPLATFORM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY ["demoapi/", "demoapi/"]
RUN dotnet restore "demoapi/demoapi.csproj"

WORKDIR "/src/demoapi"
FROM build AS publish
RUN dotnet publish "demoapi.csproj" -o /app

FROM base AS final
WORKDIR /app
COPY --from=publish /app .
ENTRYPOINT ["dotnet", "demoapi.dll"]